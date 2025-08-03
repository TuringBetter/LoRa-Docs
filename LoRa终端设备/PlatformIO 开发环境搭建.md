# PlatformIO 开发环境搭建

PlatformIO是Microsoft Visual Studio Code IDE的一款免费插件，可以完成多个嵌入式平台的开发工作。以下是该开发环境搭建的过程：

## 1、安装VS Code

安装链接如下：[VSCode](https://code.visualstudio.com/Download)，根据开发机器的配置选择对应的安装包下载安装即可。

## 2、安装Python环境

若开发机并没有配置相应环境，可点击此链接安装：[python](https://www.python.org/getit/)，同样根据电脑的配置选择最新版本的Python下载安装即可。安装后，以Windows系统为例，打开命令提示符，输入Python，若出现Python版本号则为安装成功。

## 3、安装PlatformIO插件

启动VSCode，在左边的的扩展应用中搜索platformio，点击安装即可，安装过程如果需要安装其他扩展插件，默认安装即可。提示安装成功之后，重启VScode。

![1](pics/1.png)

## 4、创建一个PlatformIO项目

安装成功后，VSCode的左侧栏目会新增一个插件图标，点击该图标，然后点击PIO Home的Open，在右侧点击新建项目New Project，填写我们的项目名称等信息。开发板Board一栏选择开发板型号，开发框架Framework选择Arduino，之后点击Finish新建项目。![2](pics/2.png)

![3](pics/3.png)

## 5、编译下载

安装了该插件后，VSCode下方的操作栏会新增一些PlatformIO的图标，作用如下图所示：

![4](pics/4.png)

编写项目工程代码，通过串口连接好开发板后，编译下载即可。