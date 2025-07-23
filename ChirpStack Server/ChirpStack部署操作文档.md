# ChirpStack部署操作文档

## Step1.克隆chirpstack v3仓库文件

```bash
git clone -b v3 https://github.com/chirpstack/chirpstack-docker.git
```

![1](pics/1.png)

```bash
cd chirpstack-docker
docker-compose up -d
```

![2](pics/2.png)

## Step2.查看运行状态

```bash
docker-compose ps
```

![3](pics/3.png)

## Step3.登录

打开部署服务器的ip 8080端口，进入ChirpStack应用程序服务器Web界面，默认账号密码都是admin。
登录后，我们可以看到如下画面：![4](pics/4.png)

## Step4.添加网络服务器

首先，添加chirpstack网络服务器，方便我们后续上行接收和下行数据。
由于部署的组件在同一docker网络下，则chirpstack应用服务器可以通过容器名`chirpstack-network-server`访问到chirpstack网络服务器。

![5](pics/5.png)

此时，网络服务器默认使用的频段是`EU868`，我们可以进入目录文件`configuration/chirpstack-network-server/chirpstack-network-server.toml`该文件用于确保网络服务器正确运行并与其他组件协同工作。根据项目的需求和所使用的LoRaWAN频段，进行相应的配置调整等。

修改网络服务器接收的频段配置：
例如中国大陆使用的频段为`CN_470_510`

```toml
[postgresql]
dsn="postgres://chirpstack_ns:chirpstack_ns@postgresql/chirpstack_ns?sslmode=disable" #这一部分配置了与PostgreSQL数据库的连接信息，包括数据库连接字符串（DSN），用户名和密码等。ChirpStack Network Server使用PostgreSQL来存储和管理网络服务器的数据。

[redis]
url="redis://redis:6379" #这一部分配置了与Redis数据库的连接信息，包括Redis的URL。Redis通常用于缓存和存储临时数据，以提高性能和响应速度。

[network_server]
net_id="000000" #这一部分包含网络服务器的一般设置，如网络标识符（Net ID）。Net ID用于标识LoRaWAN网络的唯一标识符。

[network_server.band]
name="CN_470_510" #这里指定了LoRaWAN的频段或带宽

[network_server.network_settings] #这一部分包含了有关网络设置的信息，如额外的通道配置
#额外的通道（extra_channels）被配置为特定的频率范围和数据速率范围
  # [[network_server.network_settings.extra_channels]]
  # frequency=867100000
  # min_dr=0
  # max_dr=5

  # [[network_server.network_settings.extra_channels]]
  # frequency=867300000
  # min_dr=0
  # max_dr=5

  # [[network_server.network_settings.extra_channels]]
  # frequency=867500000
  # min_dr=0
  # max_dr=5

  # [[network_server.network_settings.extra_channels]]
  # frequency=867700000
  # min_dr=0
  # max_dr=5

  # [[network_server.network_settings.extra_channels]]
  # frequency=867900000
  # min_dr=0
  # max_dr=5

[network_server.gateway.backend.mqtt]
server="tcp://mosquitto:1883" #这一部分配置了与MQTT（消息队列遥测传输）后端的连接信息。ChirpStack Network Server使用MQTT与网关通信，以接收来自网关的设备数据。

[join_server.default]#这里指定了默认的加入服务器（Join Server）的地址。加入服务器用于管理设备的入网和安全性。在这个例子中，加入服务器的地址是一个HTTP地址，用于与ChirpStack应用服务器通信。
server="http://chirpstack-application-server:8003"
```

## ChirpStack Application Server各模块功能

### Organizations

组织下管理着users、Service-profiles、Device-profiles、Gateways、Application等模块，不同的组织管理着各自的这些模块，说白了就是不同的组织下的应用分类、设备集合、网关等相互隔离，自个管着自个的数据。
例如现在有几家不同的企业承接了A园区的所有智慧厕所，A园区的技术团队掌管着这个ChirpStack服务器，A园区的管理员就可以为这几家不同的企业分别创建他们的Organization，例如有三家企业，分别为A-1、A-2、A-3：![6](pics/6.png)

我们切换到不同的企业的Organization下，就可以看到他们所管理的网关、设备、用户等。

### Organization users

每个组织可以添加自己的账号，并赋予权限。

![7](pics/7.png)

### Organization API keys

通过添加这个API key，我们可以在服务器上调用chirpstack应用服务的接口。

### Service-profiles

在创建Device-profiles、Gateways、Applications时，需要先有Service-profiles。
Service-profiles可以理解为用户和网络服务器之间的一个协同约定，通过Service-profiles可以配置启动的功能，例如启动网络地理定位等，还可以调节允许的数据速率、设备状态请求频率等。我的理解是通过这个配置文件，可以开启/限制来自网络数据的某些功能/特征。

### Device-profiles

设备配置文件，定于了接入网络服务器的设备所需的功能和参数，这些元信息由终端设备制造商提供。
主要包括：
	MAC version：终端设备支持的LoRaWAN版本
	ADR算法：控制设备数据速率
	LoRaWAN Regional Parameters revision：设备支持的区域参数规范的修订
	MaxEIRP ：终端设备支持的最大 EIRP
等。

![8](pics/8.png)

我们重点是需要关注以下参数：

**JOIN-设备加入网络的方式：**
设备需要加入网络服务器，与服务建立通道后才能支持数据的上行下行。
**OTAA**
设备通过发送入网请求（Join）给网络服务器，该消息包含设备EUI和AppKey等信息来向网络服务器验证自己的身份，验证通过后，网络服务器会为设备分配设备标识符（DevAddr）和应用会话密钥（Application Session Key）等信息用于加密和解密设备和服务器之间的通信，完成后设备与网络服务器之间即建立会话。（AppKey需要在Device配置，接下来会讲到）
![9](pics/9.png)

**ABP**
设备无需入网请求，设备通过和网络服务器之间预设配置会话参数的方式来与网络服务器进行通信。设备在制造商生产和部署前会进行配置好设备地址（DevAddr）、网络会话密钥（NwkSKey）和应用会话密钥（AppSKey）等会话参数为与网络服务器直接建立通信会话（这些DevAddr NwkSKey AppSKey需要在Device配置，接下来会讲到）。
另外需要指定制造商预设的LoRaWAN频段，例如923, 923, 923, 923, 924, 924, 924, 924

![10](pics/10.png)

**终端的工作模式：**
**Class A**
终端不上报数据，服务器就无法主动进行下行传输，要求应用在终端上传数据的很短时间内进行服务器的下行传输，服务在其他时间进行的下行传输都得等终端的下一次上行。这是最省电的通信方式，也是默认的通信方式，但接收机会最少。
**Class B**
Class B在Class A基础上添加了时间同步和额外接收端口，设备除了会在按固定的接收端口等待网络服务器的响应外，还会再预定的时间间隔内打开额外的接收端口。

![11](pics/11.png)

**Class C**
基本一直打开着接收端口，只在发送时短暂关闭。意味着设备可以实时接收网络服务器的下行消息，但也会以牺牲功耗为代价。

![12](pics/12.png)

### Gateways

这个即对应LoRa网关，例如承包园区公厕的企业在现场安装了LoRa网关来建立覆盖感应器通讯网络，那他们可以在Gateways上面创建一个虚拟的网关代表他们在现场对应的网关，并通过gatewayId与网络服务器上报的现场的gateway联系起来，可以检测网关活跃状态、接收/传送数据包数量、位置自发现’帧等信息。
![13](pics/13.png)

![14](pics/14.png)

### Applications

Application可以用来管理不同设备集合，例如承包园区智慧公厕的几家企业，肯定是分别做不同的洗手间工程，那么每个企业（相当于每个Organization）可以在其Applications下创建不同的Application分别代表当前企业承接哪些洗手间，例如A-1企业为他承包的toilets1、toilets2创建了名字为toilets1、toilets2的Application，两个Application分别代表两个洗手间。
![15](pics/15.png)

### Devices

这个Device即是在平台上模拟的对应现场的设备/感应器。

![16](pics/16.png)

创建后，当选择的device-profile为OTAA模式时，需要填写Application Key，设备入网的时候，也会上传一个Application Key，如果设备的Application Key与我们在这里给它配置的Application Key一致的话，则设备会通过信赖，为设备分配相关参数，与网络服务器建立会话。
![17](pics/17.png)

当选择的device-profile为ABP模式时，需要添加Device address、Network session key、Application session key，这个设备产商出厂时会写入设备中，需由设备厂商提供，然后往这里配置。

![18](pics/18.png)

至此，LoRaWAN服务器ChirpStack服务平台相关配置工作已做完。