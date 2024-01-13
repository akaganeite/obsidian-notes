# 名词解释
## 摘要
1. TEE
2. secure enclave
3. privileged host software attacks
4. lightweight physical attacks
5. memory access based side-channel attacks
6. side-channel attacks：侧信道攻击
## intro
1. cold boot attacks
2. bus monitoring attakcs
3. DMA attacks
4. page fault based side-channel attacks
5. cache based side-channel attacks
6. SoC-bound execution environment technology
7. page coloring technique
8. cache maintenance
9. cross-core cache attacks
10. 

# 摘要
ARM架构无法提供和secure enclave同等质量的安全机制。因此文章提出Sec-TEE，基于软件的secure enclave架构。优点是在可接受的计算开销下能够抵御privileged host software attacks, lightweight physical attacks,memory access based side-channel attacks。作者将sectee应用于ARM Trustzone技术。
## 总结 
作者在软件层面针对ARM安全性差的问题提出了解决方案，粗略地说就是Trust zone pro max。

# Intro
## secure enclaves简介
SE(secure enclave)是安全的，独立的执行环境。
A specialized memory encryption engine on the CPU encrypts DRAM regions belonging to secure enclaves, and thus no code/data is stored outside CPU in plaintext form.
在CPU中的一种特制的存储加密引擎将内存区域加密为属于SE，这样在CPU外没有数据或代码是在未加密状态下被储存的。
这样的安全措施防止CPU外的物理攻击。
## ARM的缺点
ARM架构没有SE，但是有trust zone。Trust zone为重要代码隔离出一个TEE，可信执行环境。但无法抵抗物理攻击和软件side-channel attack。
## 介绍SecTEE

> 我们通过利用基于软件的安全原语来抵抗物理攻击和侧通道攻击来实现我们的目标，并且不需要修改CPU硬件，并且只需要一些在普通CPU上常见的基本安全硬件资源

有TEE，Soc-bound 执行环境技术的加持，SecTEE有和intelsgx同样的能力；基于page coloring 技术和CPU对cache maintenance的硬件支持，SecTEE可以低于基于内存访问的side-channel攻击。
### TEE OS
SecTEE和Intel SGX的不同之处就是SecTEE拥有特制的OS，TEE OS。这个操作系统提供enclave管理，以及所有enclave的管理功能。这样可以使得enclave的内存管理与规划不暴露给系统软件，减少主机应用程序复杂度。
### security-enhanced TEE OS
SecTEE extends to the TEE OS critical trusted computing features required by secure enclaves, including enclave identification, enclave measurement, remote attestation, data sealing, secrets provisioning, and life cycle management of enclaves.
这段没看懂，后面出现新的概念，安全性增强的TEE OS，可能就是SecTEE。
enclave被载入系统前，s-e TEE OS(SecTEE)检查这个enclave完整性等等。载入后，enclave通过系统调用向外部实体验证身份，存储敏感数据。在验证之后，外部实体可将秘密存储在enclave中。

这里是说enclave和外部通信的机制是系统调用？

在enclave开发者角度，SecTEE就是个安全性增强的TEE.SecTEE实现平台：NXP i.MX6Q

# 贡献
1. 提出可以和trustzone兼容的secure enclave架构，适用于商用ARM架构CPU
2. 设计一个locking机制，将enclave的页锁住，与page coloring技术结合，低于memory based side-channel attack
3. 一种向TEE系统添加丰富可信计算特性的方法
4. 证明将SecTEE架构应用于TEE系统是可行的，计算开销增加可接受

# 背景知识
## Intel SGX
是一套CPU指令集，用于创建，运行和管理secure enclave。SGX从内存中分离出一部分内存，命名为PRM(Processor Reserved Memory)。enclave的数据和代码就存储在PRM中的Enclave Page Cache(EPC)中。SGX还有一个特制的Memory Encryption Engine(MEE)，用来预防物理攻击。
enclave被映射至主机应用中虚拟地址中的预留内存空间，被称为ELRANGE。在ELRANGE外的虚拟地址只能被映射至非enclave数据代码。
主机系统软件对enclave进行管理。系统软件给新创建的enclave分配EPC页面，建立EPC页面与ELRANGE之间的映射。
## ARM Cache结构
有两级cache，L1有独立的数据和指令cache。L2是统一的。
cache有N-ways，每个way大小相同。一个way包含k个cache行，下标0至k-1。所有way中下标相同的行组成一个cache set。内存中一块的大小等于一个cache way。每个cache有NS位，表示这一行是属于安全或者普通世界。
## Soc和Pager
SoC是片载芯片的集合，SoC-bound execution environment是抵御物理攻击的软件方式，利用SoC如CPU寄存器，cache等为重要代码设置TEE。
Pager是一个请求分页系统，用于抵御针对OP-TEE的攻击。pager在板载存储（OCM）上运行，在内存和OCM中换页，页换至OCM时解密，换出时加密。