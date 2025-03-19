前提：starry-os中每个模块单独作为一个仓库存储，这里axnet的cargo中通过feature引入了其他如底层驱动模块，可以单独作为一个网络模块进行发包测试

测试代码编写逻辑：
- bench.rs中为DeviceWrapper定义了发包和收包的测试带宽函数，并在mod.rs中进行了封装
- 发送需要定义一个网络设备，其在mod.rs中定义了ETH0，并给出了网络设备的初始化函数，后续在lib.rs中进行封装
- loopback.rs中定义了本地环回设备的初始化函数

其中没有找到现成的测试函数，需要自己实现，no_std环境下应该使用qemu进行测试
**如何定义虚拟设备并传入还有待研究**

```
pub(crate) fn init(_net_dev: AxNetDevice) {
    let mut device = LoopbackDev::new(Medium::Ip);
    let config = Config::new(smoltcp::wire::HardwareAddress::Ip);

    let mut iface = Interface::new(
        config,
        &mut device,
        Instant::from_micros_const((current_time_nanos() / NANOS_PER_MICROS) as i64),
    );
    iface.update_ip_addrs(|ip_addrs| {
        ip_addrs
            .push(IpCidr::new(IpAddress::v4(127, 0, 0, 1), 8))
            .unwrap();
    });
    LOOPBACK.init_by(Mutex::new(iface));
    LOOPBACK_DEV.init_by(Mutex::new(device));

    let ether_addr = EthernetAddress(_net_dev.mac_address().0);
    let eth0 = InterfaceWrapper::new("eth0", _net_dev, ether_addr);

    let ip = IP.parse().expect("invalid IP address");
    let gateway = GATEWAY.parse().expect("invalid gateway IP address");
    eth0.setup_ip_addr(ip, IP_PREFIX);
    eth0.setup_gateway(gateway);

    ETH0.init_by(eth0);
    info!("created net interface {:?}:", ETH0.name());
    info!("  ether:    {}", ETH0.ethernet_address());
    info!("  ip:       {}/{}", ip, IP_PREFIX);
    info!("  gateway:  {}", gateway);

    SOCKET_SET.init_by(SocketSetWrapper::new());
    LISTEN_TABLE.init_by(ListenTable::new());
}
```
以上为核心初始化函数，其定义了回环设备，并将本地设备传入封装为可被smoltcp识别的设备ETH0

而bench.rs中的函数因为直接发送以太网帧，直接在数据链路层发送数据，不经过 TCP/UDP 协议栈，理论上接近网卡

学长ppt中实际tcp发送的测试环境不太了解