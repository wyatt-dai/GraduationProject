整体框架分为5个模块，并不是完全的类似osi模型的上下级关系  
- socket：最顶层与应用对接，负责提供不同协议的统一接口，如tcp,udp,icmp,dns等，这里封装的是高层的细节，而wire负责较底层的数据包产生
- iface:直接与网络设备交互的接口层,整合wire和socket模块中函数
- storage：静态的缓冲区实现，未关注实现细节
- wire：负责各个协议的底层数据包产生，以及接受时候的解析,因为主要是负责数据报封装的接口，个人理解没有太多可优化空间，没有太多关注实现细节
- phy：负责最底层的虚拟设备传输，如loopback,rawsocket,和tuntap传输



### 一. socket模块：
主要阅读了tcp的实现，其中实现了阻塞控制算法等
- 如timer结构体，负责管理在滑动窗口协议中的传输时间管理，这里使用poll_at传输完成执行
- **socket结构体**，用于维护单个连接，实现的重要函数ack_delay,listen，bind,send等
```
pub struct Socket<'a> {//对每一个连接维护一个socket结构体，后续使用socket_set来整体管理
    state: State,
    timer: Timer,
    rtte: RttEstimator,
    assembler: Assembler,
    rx_buffer: SocketBuffer<'a>,
    rx_fin_received: bool,
    tx_buffer: SocketBuffer<'a>,
    /// Interval after which, if no inbound packets are received, the connection is aborted.
    timeout: Option<Duration>,
    /// Interval at which keep-alive packets will be sent.
    keep_alive: Option<Duration>,
    /// The time-to-live (IPv4) or hop limit (IPv6) value used in outgoing packets.
    hop_limit: Option<u8>,
    /// Address passed to listen(). Listen address is set when listen() is called and
    /// used every time the socket is reset back to the LISTEN state.
    listen_endpoint: IpListenEndpoint,
    /// Current 4-tuple (local and remote endpoints).
    tuple: Option<Tuple>,//发送接受节点
    /// The sequence number corresponding to the beginning of the transmit buffer.
    /// I.e. an ACK(local_seq_no+n) packet removes n bytes from the transmit buffer.
    local_seq_no: TcpSeqNumber,
    /// The sequence number corresponding to the beginning of the receive buffer.
    /// I.e. userspace reading n bytes adds n to remote_seq_no.
    remote_seq_no: TcpSeqNumber,
    /// The last sequence number sent.
    /// I.e. in an idle socket, local_seq_no+tx_buffer.len().
    remote_last_seq: TcpSeqNumber,
    /// The last acknowledgement number sent.
    /// I.e. in an idle socket, remote_seq_no+rx_buffer.len().
    remote_last_ack: Option<TcpSeqNumber>,
    /// The last window length sent.
    remote_last_win: u16,
    /// The sending window scaling factor advertised to remotes which support RFC 1323.
    /// It is zero if the window <= 64KiB and/or the remote does not support it.
    remote_win_shift: u8,
    /// The remote window size, relative to local_seq_no
    /// I.e. we're allowed to send octets until local_seq_no+remote_win_len
    remote_win_len: usize,
    /// The receive window scaling factor for remotes which support RFC 1323, None if unsupported.
    remote_win_scale: Option<u8>,
    /// Whether or not the remote supports selective ACK as described in RFC 2018.
    remote_has_sack: bool,
    /// The maximum number of data octets that the remote side may receive.
    remote_mss: usize,
    /// The timestamp of the last packet received.
    remote_last_ts: Option<Instant>,
    /// The sequence number of the last packet received, used for sACK
    local_rx_last_seq: Option<TcpSeqNumber>,
    /// The ACK number of the last packet received.
    local_rx_last_ack: Option<TcpSeqNumber>,
    /// The number of packets received directly after
    /// each other which have the same ACK number.
    local_rx_dup_acks: u8,

    /// Duration for Delayed ACK. If None no ACKs will be delayed.
    ack_delay: Option<Duration>,
    /// Delayed ack timer. If set, packets containing exclusively
    /// ACK or window updates (ie, no data) won't be sent until expiry.
    ack_delay_timer: AckDelayTimer,

    /// Used for rate-limiting: No more challenge ACKs will be sent until this instant.
    challenge_ack_timer: Instant,

    /// Nagle's Algorithm enabled.
    nagle: bool,

    /// The congestion control algorithm.
    congestion_controller: congestion::AnyController,

    /// tsval generator - if some, tcp timestamp is enabled
    tsval_generator: Option<TcpTimestampGenerator>,

    /// 0 if not seen or timestamp not enabled
    last_remote_tsval: u32,

    #[cfg(feature = "async")]
    rx_waker: WakerRegistration,
    #[cfg(feature = "async")]
    tx_waker: WakerRegistration,
}
```
其中send的功能是将想要发送的内容写入到tcp_socket的buffer中，此时还处于待处理状态，未封装成帧

---

### 二.wire模块：
主要实现的就是packet结构体，本身是个buffer，通过为其实现的结构体方法进行帧的生成

---

### 三.phy模块：
在mod文件中主要定义了device trait， loopback.rs, tuntap_interface.rs等，基本就是对这个trait的实现
```
pub trait Device {
    type RxToken<'a>: RxToken
    where
        Self: 'a;
    type TxToken<'a>: TxToken
    where
        Self: 'a;

    /// Construct a token pair consisting of one receive token and one transmit token.
    ///
    /// The additional transmit token makes it possible to generate a reply packet based
    /// on the contents of the received packet. For example, this makes it possible to
    /// handle arbitrarily large ICMP echo ("ping") requests, where the all received bytes
    /// need to be sent back, without heap allocation.
    ///
    /// The timestamp must be a number of milliseconds, monotonically increasing since an
    /// arbitrary moment in time, such as system startup.
    fn receive(&mut self, timestamp: Instant) -> Option<(Self::RxToken<'_>, Self::TxToken<'_>)>;

    /// Construct a transmit token.
    ///
    /// The timestamp must be a number of milliseconds, monotonically increasing since an
    /// arbitrary moment in time, such as system startup.
    fn transmit(&mut self, timestamp: Instant) -> Option<Self::TxToken<'_>>;

    /// Get a description of device capabilities.
    fn capabilities(&self) -> DeviceCapabilities;
}
```
其中**RxToken**和**TxToken**两个令牌是发送和接受函数返回的对象，也就是实际的发送需要等到调用了TxToken的**consume**方法，而读取也是调用consume后复制到缓冲区，最终poll的实现才完成整个发送接受操作。
设备capability结构体如下，功能比较显然
```
pub struct DeviceCapabilities {
    pub medium: Medium,//传输介质，决定使用的协议处理策略
    pub max_transmission_unit: usize,
    pub max_burst_size: Option<usize>,
    pub checksum: ChecksumCapabilities,//是否启用
}
```

---

### 四.iface模块：
定义了interface结构体和响应的poll函数，其中interface维护一些封装和发送等处理所需的信息
```
pub struct Interface {
    pub(crate) inner: InterfaceInner,
    fragments: FragmentsBuffer,
    fragmenter: Fragmenter,
}

pub struct InterfaceInner {
    caps: DeviceCapabilities,
    now: Instant,
    rand: Rand,

    #[cfg(any(feature = "medium-ethernet", feature = "medium-ieee802154"))]
    neighbor_cache: NeighborCache,
    hardware_addr: HardwareAddress,
    #[cfg(feature = "medium-ieee802154")]
    sequence_no: u8,
    #[cfg(feature = "medium-ieee802154")]
    pan_id: Option<Ieee802154Pan>,
    #[cfg(feature = "proto-ipv4-fragmentation")]
    ipv4_id: u16,
    #[cfg(feature = "proto-sixlowpan")]
    sixlowpan_address_context:
        Vec<SixlowpanAddressContext, IFACE_MAX_SIXLOWPAN_ADDRESS_CONTEXT_COUNT>,
    #[cfg(feature = "proto-sixlowpan-fragmentation")]
    tag: u16,
    ip_addrs: Vec<IpCidr, IFACE_MAX_ADDR_COUNT>,
    any_ip: bool,
    routes: Routes,
    #[cfg(feature = "multicast")]
    multicast: multicast::State,
}
```
**poll函数**
触发封装和发送的方法，调用了socket_ingress和socket_egress分别处理发送和接收
```
pub fn poll(
        &mut self,
        timestamp: Instant,
        device: &mut (impl Device + ?Sized),
        sockets: &mut SocketSet<'_>,
    ) -> PollResult {
        self.inner.now = timestamp;

        let mut res = PollResult::None;

        #[cfg(feature = "_proto-fragmentation")]
        self.fragments.assembler.remove_expired(timestamp);

        // Process ingress while there's packets available.
        loop {
            match self.socket_ingress(device, sockets) {
                PollIngressSingleResult::None => break,
                PollIngressSingleResult::PacketProcessed => {}
                PollIngressSingleResult::SocketStateChanged => res = PollResult::SocketStateChanged,
            }
        }

        // Process egress.
        match self.poll_egress(timestamp, device, sockets) {
            PollResult::None => {}
            PollResult::SocketStateChanged => res = PollResult::SocketStateChanged,
        }

        res
    }
```
**接收函数**
比较简单，通过设备receive讲数据包从硬件缓冲区获取为（txtoken,rxtoken），然后进行处理

**发送函数**
```
fn socket_egress(
        &mut self,
        device: &mut (impl Device + ?Sized),
        sockets: &mut SocketSet<'_>,
    ) -> PollResult {
        let _caps = device.capabilities();

        enum EgressError {
            Exhausted,
            Dispatch,
        }

        let mut result = PollResult::None;
        for item in sockets.items_mut() {//遍历所有socket
            if !item
                .meta
                .egress_permitted(self.inner.now, |ip_addr| self.inner.has_neighbor(&ip_addr))
            {
                continue;
            }

            let mut neighbor_addr = None;
            let mut respond = |inner: &mut InterfaceInner, meta: PacketMeta, response: Packet| {//定义ip分片闭包
                neighbor_addr = Some(response.ip_repr().dst_addr());
                let t = device.transmit(inner.now).ok_or_else(|| {
                    net_debug!("failed to transmit IP: device exhausted");
                    EgressError::Exhausted
                })?;

                inner
                    .dispatch_ip(t, meta, response, &mut self.fragmenter)
                    .map_err(|_| EgressError::Dispatch)?;

                result = PollResult::SocketStateChanged;

                Ok(())
            };

            let result = match &mut item.socket {//区分协议分别进行数据包构建
                #[cfg(feature = "socket-raw")]
                Socket::Raw(socket) => socket.dispatch(&mut self.inner, |inner, (ip, raw)| {
                    respond(
                        inner,
                        PacketMeta::default(),
                        Packet::new(ip, IpPayload::Raw(raw)),
                    )
                }),
                #[cfg(feature = "socket-icmp")]
                Socket::Icmp(socket) => {
                    socket.dispatch(&mut self.inner, |inner, response| match response {
                        #[cfg(feature = "proto-ipv4")]
                        (IpRepr::Ipv4(ipv4_repr), IcmpRepr::Ipv4(icmpv4_repr)) => respond(
                            inner,
                            PacketMeta::default(),
                            Packet::new_ipv4(ipv4_repr, IpPayload::Icmpv4(icmpv4_repr)),
                        ),
                        #[cfg(feature = "proto-ipv6")]
                        (IpRepr::Ipv6(ipv6_repr), IcmpRepr::Ipv6(icmpv6_repr)) => respond(
                            inner,
                            PacketMeta::default(),
                            Packet::new_ipv6(ipv6_repr, IpPayload::Icmpv6(icmpv6_repr)),
                        ),
                        #[allow(unreachable_patterns)]
                        _ => unreachable!(),
                    })
                }
                #[cfg(feature = "socket-udp")]
                Socket::Udp(socket) => {
                    socket.dispatch(&mut self.inner, |inner, meta, (ip, udp, payload)| {
                        respond(inner, meta, Packet::new(ip, IpPayload::Udp(udp, payload)))
                    })
                }
                #[cfg(feature = "socket-tcp")]
                Socket::Tcp(socket) => socket.dispatch(&mut self.inner, |inner, (ip, tcp)| {
                    respond(
                        inner,
                        PacketMeta::default(),
                        Packet::new(ip, IpPayload::Tcp(tcp)),
                    )
                }),
                #[cfg(feature = "socket-dhcpv4")]
                Socket::Dhcpv4(socket) => {
                    socket.dispatch(&mut self.inner, |inner, (ip, udp, dhcp)| {
                        respond(
                            inner,
                            PacketMeta::default(),
                            Packet::new_ipv4(ip, IpPayload::Dhcpv4(udp, dhcp)),
                        )
                    })
                }
                #[cfg(feature = "socket-dns")]
                Socket::Dns(socket) => socket.dispatch(&mut self.inner, |inner, (ip, udp, dns)| {
                    respond(
                        inner,
                        PacketMeta::default(),
                        Packet::new(ip, IpPayload::Udp(udp, dns)),
                    )
                }),
            };

            match result {
                Err(EgressError::Exhausted) => break, // Device buffer full.
                Err(EgressError::Dispatch) => {
                    // `NeighborCache` already takes care of rate limiting the neighbor discovery
                    // requests from the socket. However, without an additional rate limiting
                    // mechanism, we would spin on every socket that has yet to discover its
                    // neighbor.
                    item.meta.neighbor_missing(
                        self.inner.now,
                        neighbor_addr.expect("non-IP response packet"),
                    );
                }
                Ok(()) => {}
            }
        }
        result
    }
}
```

---

### 整体的运行流程：
1.初始化设备，socket,interface  
2.socket绑定，监听，建立连接  
3.socket调用send函数将数据发送到buffer  
4.poll函数会处理发送和接收，其中接收是放入缓冲区  
5.socket调用recv接收  
