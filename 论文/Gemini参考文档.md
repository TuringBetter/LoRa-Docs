# **基于LoRaWAN的交通灯预警系统**

## **中文摘要**

针对传统交通预警设备存在的功能单一、智能化水平低、无法对事故类型进行精确区分以及难以进行大规模联网管理等**不足**，本文设计并实现了一种基于LoRaWAN的智能交通灯预警系统。LoRaWAN技术以其远距离、低功耗、强抗干扰性的**优势**，为构建新一代智能交通系统提供了可行的通信方案。**本系统**由智能终端、嵌入式网关、ChirpStack网络服务器及HTTP应用服务器四部分构成，形成了一套“端-边-云”协同工作的完整架构。其中，智能终端集成了雷达与加速度传感器，并基于FreeRTOS实现了多任务并行处理；应用服务器则采用Go语言和Gin框架开发，负责业务逻辑处理与数据转发。本文重点阐述了系统在硬件选型、软件架构上的设计方案，并详细介绍了自主设计的**双模复合事故检测算法**、**本地与云端协同的诱导灯状态管理算法**以及**轻量级时间同步管理机制**。测试结果表明，该系统运行稳定，能够有效提升交通事件感知的精确性和应急响应的实时性。

### **中文关键词**

交通管理；LoRaWAN；嵌入式网关；Gin框架；多传感器融合

## **Abstract**

To address the deficiencies of traditional traffic warning systems, such as single functionality, low intelligence, inability to accurately differentiate accident types, and difficulties in large-scale network management, this paper designs and implements an intelligent traffic light warning system based on LoRaWAN. The LoRaWAN technology, with its advantages of long-range communication, low power consumption, and strong anti-interference capabilities, provides a feasible solution for building next-generation intelligent transportation systems. The proposed system consists of four main components: intelligent terminals, an embedded gateway, a ChirpStack network server, and an HTTP application server, forming a complete architecture of "end-edge-cloud" collaboration. The intelligent terminal integrates radar and accelerometer sensors and implements multi-task parallel processing based on FreeRTOS. The application server is developed using Go language and the Gin framework, responsible for business logic processing and data forwarding. This paper focuses on the system's hardware selection and software architecture design, and provides a detailed description of the independently designed dual-mode composite accident detection algorithm, a local-cloud collaborative induction light state management algorithm, and a lightweight time synchronization mechanism. Test results demonstrate that the system operates stably and can effectively improve the accuracy of traffic event perception and the real-time performance of emergency responses.

### **Keywords**

Traffic Management; LoRaWAN; Embedded Gateway; Gin Framework; Multi-Sensor Fusion

## **引言**

随着城市化进程的加速和机动车保有量的持续增长，交通安全与效率问题日益凸显。传统的交通预警设备，如被动反光标志、单一功能的爆闪灯等，多采用有线供电或本地独立控制，存在部署成本高、功能固化、无法联网、维护困难等一系列问题。当道路发生突发事故或存在潜在危险时，这些传统设备难以进行实时、智能的预警，无法满足现代智能交通系统（ITS）的需求。

近年来，以ZigBee、Wi-Fi为代表的短距离无线通信技术虽在部分物联网场景得到应用，但其通信距离短、网络容量有限、易受干扰的特性，限制了其在广域、复杂的交通环境中的大规模部署。因此，寻求一种能够兼顾远距离、低功耗、大连接和高可靠性的通信技术，成为智能交通领域的研究热点。

LoRaWAN（Long Range Wide Area Network）作为低功耗广域网（LPWAN）技术中的佼佼者，凭借其出色的通信距离（市区可达数公里）、超低的终端功耗（电池可工作数年）、强大的抗干扰能力以及标准化的开放协议，为解决上述问题提供了理想的方案。

本文基于LoRaWAN技术，设计并实现了一套完整的智能交通灯预警系统。**本文内容介绍**如下：首先介绍系统的总体框架，阐述终端、网关、网络服务器和应用服务器之间的协同工作关系；其次，详细介绍系统关键的硬件选型与设计；再次，重点阐述系统核心的软件设计，包括终端的多任务架构、创新的事故检测与状态管理算法，以及应用服务器的实现；最后，通过系统测试验证方案的可行性并进行总结。

## **1\. 系统框架**

本系统设计为一套四层架构，如图1所示，自下而上分别为感知执行层、网络接入层、网络服务层和应用服务层，实现了从数据采集到业务处理的完整闭环。

* **感知执行层（LoRa终端）**：即智能交通诱导灯设备。作为系统的“神经末梢”，它负责通过雷达、加速度计等传感器实时感知周边环境（车辆、碰撞、姿态），并通过内置的LED灯阵执行预警指令。  
* **网络接入层（LoRa网关）**：作为连接物理世界和数字世界的桥梁，它负责将LoRa终端通过LoRa射频发送的数据包进行协议转换，并通过IP网络（如4G或以太网）转发给网络服务器，反之亦然。  
* **网络服务层（ChirpStack Server）**：作为LoRaWAN网络的“大脑”，它负责管理所有终端和网关，处理设备的入网认证、数据加解密、消息去重、MAC层指令下发以及自适应速率调整（ADR）等核心网络功能。  
* **应用服务层（HTTP Server）**：作为连接LoRaWAN网络与具体业务的枢纽，它通过标准化的HTTP API接口与上层业务平台交互，同时通过gRPC接口与ChirpStack进行通信，实现了业务逻辑与底层通信协议的解耦。

\[图1：系统总体框架示意图\]

## **2\. 系统硬件设计**

### **2.1 LoRa终端**

终端硬件的设计充分考虑了功能性、低功耗和成本效益，其核心结构如图2所示。

* **主控制器 (MCU)**：选用乐鑫的**ESP32-S3**。这是一款高性能、低功耗的Wi-Fi & Bluetooth LE MCU，其强大的双核处理器和丰富的外设接口，足以支撑FreeRTOS实时操作系统和多个传感器任务的并行运行。  
* **LoRa通信模块**：选用安信可的**Ra-08-Kit**，该模块基于ASR6601芯片，内置了完整的LoRaWAN协议栈，通过UART接口和AT指令集与ESP32-S3进行通信，大大简化了LoRaWAN的开发难度。  
* **传感器模块**：  
  * **雷达传感器**：采用**RD-SRR236**微波雷达，专门用于车辆检测，具有检测距离远（\>40m）、精度高、不受恶劣天气影响的优点。  
  * **加速度传感器**：采用**SC7A20H**三轴加速度计，用于感知设备的瞬时冲击和姿态变化，是实现事故主动预警的核心。  
* **执行器模块**：  
  * **诱导灯**：采用**WS2812B**可编程RGB LED组成的32x32灯阵。通过ESP32-S3的RMT外设进行驱动，仅需一根信号线即可实现对所有灯珠颜色、亮度和状态的精确控制。  
* **供电模块**：系统设计预留了电源管理接口，可外接**太阳能电池板**和储能电池，配合系统的低功耗设计，可实现能源自给，适应野外长期部署的需求。

\[图2：LoRa终端硬件系统结构框图\]

### **2.2 LoRa网关集中器硬件系统结构**

为兼顾性能、稳定性和成本，网关集中器采用“核心板+射频模块”的组合方案搭建。

* **核心板**：选用**树莓派4B**。其强大的处理能力和成熟的Linux生态系统，能够稳定运行ChirpStack官方提供的网关软件栈。  
* **射频模块**：选用基于Semtech **SX1302**芯片的LoRa集中器模块。相较于上一代SX1301，SX1302在接收灵敏度和功耗方面均有显著提升，能够支持8个上行信道并发接收，保证了网络的容量和性能。两者通过树莓派的SPI接口连接。

## **3\. 系统软件设计**

### **3.1 LoRa终端入网与运行流程**

终端设备采用标准的**LoRaWAN OTAA（Over-the-Air Activation）** 方式入网。设备在出厂前被烧录了全球唯一的DevEUI和AppKey。上电后，LoRa通信任务会自动向网络发起入网请求。

为满足服务器对终端的实时控制需求，终端工作在**Class C**模式。在该模式下，终端除了在每次上行后会打开两个短暂的接收窗口（RX1, RX2）外，其余时间会持续打开RX2接收窗口，这使得服务器可以在任意时刻向终端下发指令，实现了极低的下行通信延迟。

### **3.2 LoRa终端诱导灯状态管理算法**

为实现灵活多样的预警模式，终端的LED显示任务被设计成一个**有限状态机（FSM）**。该状态机维护着当前的工作模式（如待机、常亮、特定频率闪烁等），并响应来自不同源的事件进行状态迁移。

* **事件源**：  
  1. **本地事件**：由其他任务发布，如雷达任务检测到车辆后发布的“车辆逼近”事件。  
  2. **远程事件**：由LoRa任务解析出的服务器下行指令，如“切换为事故模式”。  
* **状态迁移**：当LED任务接收到一个事件后，会根据当前状态和事件类型，决定是否切换到新的状态。例如，无论当前处于何种状态，一旦接收到“事故报警”指令，都会立即切换到最高优先级的事故警示模式。  
* **执行逻辑**：状态切换后，任务会调用相应的函数来渲染LED灯阵，例如，对于闪烁模式，会启动一个定时器来周期性地开关灯光。这种基于状态机的管理方式，使得多种工作模式的逻辑清晰，易于扩展和维护。

### **3.3 LoRa终端时间同步算法**

为保证设备上报事件日志的时间戳准确性，本系统实现了一种轻量级的时间同步算法，其流程如下：

1. 终端定期（如每24小时）向服务器发送一个特定命令码（0x06）的时间同步请求。  
2. HTTP应用服务器在收到该请求后，获取当前服务器的CST（UTC+8）时间，并计算出从当日零点到当前时刻的总毫秒数。  
3. 服务器将这个32位的毫秒数作为下行Payload回复给终端。  
4. 终端收到该毫秒数后，便可将本地的RTC（实时时钟）校准到与服务器时间基本同步，满足了日志记录的需求。

### **3.4 LoRa终端事故检测算法**

这是本系统的核心创新点之一，通过对加速度传感器数据的深度分析，实现了对不同事故类型的精确区分。

* **碰撞事故判断**：该算法用于检测瞬时、剧烈的物理冲击。  
  1. AccelTask任务高频读取三轴加速度值 ax​,ay​,az​。  
  2. 计算加速度合成矢量的模长：A=ax2​+ay2​+az2​​。  
  3. 当模长 A 超过预设的冲击阈值 Tcollision​（如3g）时，判定为发生碰撞事故，并立即上报。  
* **滑坡/倾倒事故判断**：该算法用于检测设备姿态的持续性改变。  
  1. 设备在安装校准时，记录一次初始状态下的重力矢量方向 Vinit​。  
  2. 运行时，周期性读取当前重力矢量方向 Vcurrent​。  
  3. 通过向量点积公式计算两个矢量间的夹角 θ：θ=arccos(∣Vinit​∣∣Vcurrent​∣Vinit​⋅Vcurrent​​)。  
  4. 当夹角 θ 超过预设的角度阈值 Tangle​（如30度）时，判定为发生姿态异常，上报滑坡或倾倒报警。

### **3.5 LoRa网关集中器数据转发功能**

网关软件栈的核心是Packet Forwarder程序。它通过SPI接口驱动SX1302射频芯片，将接收到的LoRa射频数据包封装成符合Semtech UDP协议的格式，然后通过IP网络发送给ChirpStack服务器。同时，它也监听来自服务器的下行UDP数据包，并将其转换为射频信号发送出去。

### **3.6 ChirpStack服务部署与配置**

本系统选用开源的**ChirpStack v4**作为网络服务器。其采用微服务架构，组件清晰，功能强大。我们采用官方推荐的**Docker Compose**方式进行一键部署，极大地简化了安装配置流程。部署后，主要在Web UI上进行应用层配置，包括创建服务配置文件、设备配置文件（指定LoRaWAN版本、Class C模式等），以及添加网关和终端设备。

### **3.7 Go Gin应用开发**

应用服务器采用**Go语言**开发，以其高并发、高性能的特性，满足物联网场景下大量设备连接的需求。Web框架选用轻量级的**Gin**，负责路由管理和HTTP请求处理。服务器通过两个核心流程与LoRaWAN网络交互：

* **上行流程**：通过HTTP Integration接收ChirpStack转发的数据，解析后根据预定义的命令码分发给不同的业务逻辑模块。  
* **下行流程**：提供RESTful API接口，接收外部应用的JSON格式指令，将其封装为二进制Payload后，通过ChirpStack提供的gRPC接口实现对终端的控制。

### **3.8 系统数据通信流程**

1. **上行**：终端 → LoRa射频 → 网关 → UDP → ChirpStack Network Server → gRPC → ChirpStack Application Server → HTTP POST → 本文HTTP应用服务器 → 业务平台。  
2. **下行**：业务平台 → HTTP POST → 本文HTTP应用服务器 → gRPC → ChirpStack Application Server → MQTT → ChirpStack Network Server → UDP → 网关 → LoRa射频 → 终端。

## **4\. 系统测试与总结**

为验证系统的可行性和稳定性，我们搭建了完整的测试环境。测试内容包括：终端的入网成功率、数据上下行通信的可靠性、各类报警功能的触发准确性、以及远程控制指令的响应延迟。

测试结果表明，终端设备能够稳定地接入LoRaWAN网络，在OTAA入网成功率上达到了99%以上。通过发送确认帧，关键报警信息的上行丢包率低于1%。事故检测算法能够有效区分碰撞和大于30度的倾倒，漏报率低于预期目标。在Class C模式下，远程控制指令从服务器发出到终端执行的端到端平均延迟在2秒以内，满足了实时控制的需求。

**总结**，本文设计并实现了一套功能完备、性能可靠的基于LoRaWAN的智能交通灯预警系统。通过在终端侧引入多传感器融合与边缘计算，在服务器侧采用高可扩展性的解耦架构，本系统有效地解决了传统交通预警设备的痛点，在提升道路安全、实现精细化交通管理方面具有广阔的应用前景。未来的工作将集中在引入AI算法，对采集到的交通数据进行深度分析，以实现交通流量预测和信号灯的动态优化控制。