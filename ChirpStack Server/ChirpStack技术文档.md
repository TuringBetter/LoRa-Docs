

# ChirpStack技术文档

## 1.总体介绍

### 1.1 LoRaWAN协议

&emsp;&emsp;LoRaWAN（长程广域网，Long Range Wide Area Network）是一种低功耗、广域覆盖的无线通信协议，专为物联网（IoT）应用而设计。它采用低功耗、长距离通信技术，使得设备可以在低功耗条件下远距离传输数据，适用于各种用例，如智能城市、农业、工业自动化、环境监测等。它基于LoRa调制技术来实现的。
### 1.2 ChirpStack介绍

&emsp;&emsp;ChirpStack是一个开源的物联网（IoT）网络服务器，它提供了LoRaWAN协议的网络服务器功能。
&emsp;&emsp;具体来说，ChirpStack是一个LoRaWAN网络服务器，用于管理和控制连接到LoRaWAN网络的终端设备、网关和应用程序。它允许用户建立、操作和维护自己的LoRaWAN网络，以支持各种物联网应用，如智能城市、农业、工业自动化和环境监测等。

### 1.3 开源的LoRaWan Server

当前开源的LoRaWan Server主要有：chirpstack、lorawan-server和ttn。
其中chirpStack和ttn是Golang实现，lorawan-server是Erlang实现。

## 2.各组件介绍

### 2.1 chirpstack v3

#### chirpstack-network-server

&emsp;&emsp;[ChirpStack网络服务器](https://github.com/brocaar/chirpstack-network-server)是LoRaWAN网络中的核心组件，负责处理网关接收到的设备的上行链路数据以及调度下行链路数据，即处理设备的注册和数据传输。它是整个LoRaWAN网络的中央协调器。


#### chirpstack-application-server

&emsp;&emsp;[ChirpStack应用服务器](https://github.com/brocaar/chirpstack-application-server)是用于管理LoRaWAN应用的组件。它负责处理设备数据、解码Payload、处理应用逻辑、提供API以与应用程序集成（支持通过Web界面或API与应用服务器交互）等。


#### chirpstack-gateway-bridge

&emsp;&emsp;ChirpStack网关桥接器是用于将网关传输的数据转发到ChirpStack网络服务器的组件，可以将LoRa网关传输来的LoRa数据包转发器协议转换成ChirpStack网络服务器通用的数据格式（JSON和Protobuf）。它充当网关和网络服务器之间的中间件。

&emsp;&emsp;LoRa网关的数据是加密的，我们在LoRa gateway log里不能具体看到数据样式，此时ChirpStack网关桥接器就可以将这种数据包转换成ChirpStack识别的数据格式，这也有利于用户的可观性。

&emsp;&emsp;chirpstack-gateway-bridge有两种部署方式，一种是内置在网关内，另一种是ChirpStack网络服务一样和部署在服务器上。

#### postgresql

&emsp;&emsp;ChirpStack使用了PostgreSQL数据库来存储ChirpStack的各种数据，包括设备注册信息、设备数据等。

#### redis

&emsp;&emsp;用于存储ChirpStack的临时数据和缓存信息。

#### mosquitto

&emsp;&emsp;Mosquitto是一个MQTT协议的代理服务器，用于处理设备和ChirpStack组件之间的通信。它允许LoRaWAN网关和ChirpStack服务之间的消息传递。

&emsp;&emsp;在LoRaWAN网关的数据包转发器配置可以选择MQTT for Chirpstack协议，给予其MQTT Broker Address MQTT Broker Port即可与ChirpStack服务进行通讯，即LoRaWAN网关与ChirpStack服务之间的通讯可以通过MQTT协议来传输。

### 2.2 chirpstack v4

#### chirpstack

ChirpStack v4是ChirpStack的最新版本，它包括网络服务器、应用服务器和设备管理。它是一个综合的LoRaWAN解决方案，允许您构建和管理LoRaWAN网络。其合并了ChirpStack v3的chirpstack-network-server和chirpstack-application-server，因此不需要在chirpstack-application-server上配置指定chirpstack-network-server。

https://github.com/chirpstack/chirpstack

#### chirpstack-gateway-bridge-eu868

同chirpstack v3
https://github.com/chirpstack/chirpstack-gateway-bridge

#### postgres、redis、mosquitto等

同chirpstack v3

### 2.3 组件间的关系

![19](pics/19.png)

&emsp;&emsp;LoRa Gateway可通过两种方式与ChirpStack交互，一种是在LoRa Gateway内置的ChirpStack Gateway Bridge转换数据格式并通过MQTT协议与MQTT broker交互，另一种是在服务器上单独部署ChirpStack Gateway Bridge，LoRa Gateway通过UDP协议与ChirpStack Gateway Bridge交互，ChirpStack Gateway Bridge再将转换完的数据与MQTT broker交互。这样子，ChirpStack Network Server即可直接与MQTT broker通过发布订阅消息来间接与LoRa Gateway实现通讯。

&emsp;&emsp;ChirpStack Application Server与ChirpStack Network Server之间通过gRPC远程调用协议进行交互，例如接收ChirpStack Network Server收集的上行数据，向ChirpStack Network Server下行payload。

&emsp;&emsp;ChirpStack Application Server可与其他外部组件集成，例如与外部MQTT broker集成，将数据发送到该MQTT broker上，通过HTTP的方式将数据通过POST请求发送到用户配置的端点的接口上，将数据写入时序数据库InfluxDB和关系型数据库PostgreSQL等。