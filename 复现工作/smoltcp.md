运行环境：
- AMD R9 7845HX CPU
- ubuntu 24.04 lts

在进行测试程序执行前需要进行tap的配置，否则需要在sudo环境下执行
以下是配置过程，按照readme中指导进行，具体指令就是创建一个tap（虚拟网络适配器），并为其添加各个节点的路由
![图片](https://github.com/user-attachments/assets/49e4e704-11ca-48ac-9b5e-0de6901638ea)
- 这里提示fe80::/64已经配置路由，检查后发现路由存在没有仔细处理，导致后续错误

![图片](https://github.com/user-attachments/assets/53169152-7d62-402c-86fb-d60b065c5a50)
以上为启动tap0后检查措施，尖括号中最后一个是up表示处于启动状态  
![图片](https://github.com/user-attachments/assets/f074191b-391e-4424-8a68-2f7d29cf3e75)
testbench的执行结果，
![图片](https://github.com/user-attachments/assets/612eff42-9082-4ae4-b785-71006dadf1ef)
![图片](https://github.com/user-attachments/assets/43eade36-4eb1-4e36-bdd3-92c40c124578)
![图片](https://github.com/user-attachments/assets/981bb876-f65f-4f1d-af30-71780cea3a43)
![图片](https://github.com/user-attachments/assets/c8b119ca-f520-4dd6-afb9-28d594ae1e3c)
