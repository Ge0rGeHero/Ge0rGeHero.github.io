---
title: 基于Overflow搭建的骰娘
date: 2024-06-10 00:23:39
tags:

---

# Overflow是什么

[Overflow](https://mirai.mrxiaom.top/)是 mirai 对于 Onebot 的转接器、适配器，桥接在 mirai 接口与 Onebot 实现中，使得用户可以在大部分标准 Onebot 实现上使用 mirai 插件 (或其它业务实现)。  
由于支持使用mirai插件，那便可使用[Dice!](https://v2docs.kokona.tech/zh/latest/index.html#)插件。

具体方法如下。

---

首先要选择你喜欢的Onebot协议，Overflow的[用户手册](https://mirai.mrxiaom.top/docs/UserManual.html)中提供了几种Onebot协议，由于我是在1.5G的Orangepi Zero3上部署的，为了方便与节约内存，所以选择[NapCatQQ](https://napneko.github.io/zh-CN/guide/getting-started)。  

1. 安装QQ  
   前往[QQ官网](https://im.qq.com/linuxqq/index.shtml)下载对应版本的linuxqq并进行安装。  

2. 下载并安装NapCatQQ  
   前往[NapCatQQ的github](https://github.com/NapNeko/NapCatQQ/releases)上下载相应架构的NapCat压缩包并进行解压。  
   `unzip NapCat.linux.arm64.zip && cd NapNeko.linux.arm64`  

3. 运行NapCat  
   `chmod u+x ./napcat.sh`  
   `./napcat.sh`  
   此时，终端窗口会出现一个二维码，扫码登录后便可Ctrl+c退出进程。之后如需再次登录相同账号，可通过`./napcat.sh -q <你的QQ>`进行快速登录而不用扫码。  

4. 配置NapCat  
   进入config文件夹，打开onebot11_<你的QQ号>，将“ws”（即为正向websocket服务）中的“enable”改为“true”。  
   
   ```
   "ws":{  
   "enable": true,
   "host": "",
   "port":3001
   },
   ```

5. 下载整合包  
   在[官网](https://mirai.mrxiaom.top/)下载最新版本的整合包并进行解压缩。  

6. 运行overflow  
   进入overflow目录，`sudo bash start.sh`  

7. 安装Dice插件  
   `git clone --depth=1 https://gitee.com/diceki/mirai-dice-classic`  
   下载整合包，将mirai-dice-classic文件夹中的Diceki复制到overflow的根目录，plugins中的mirai-native-2.0.5-cp.jar复制到overflow的相应文件夹。  
   
   ```
   cp Diceki path/to/overflow/
   cp plugins/mirai-native-2.0.5-cp.jar path/to/overflow/plugins/
   ```

8. 修改overflow配置  
   将overflow目录下的overflow.json里“ws_host”改为“<你的IP>:<ws服务的端口>”，如：`"ws_host": "ws://127.0.0.1:3001"`  

9. 最后一步  
   先启动NapCat登录QQ,再启动overflow进行连接。刚装完插件第一次登录时会安装些组件，需要等一段时间。  
   `./napcat.sh`  
   `sudo bash start.sh`  
