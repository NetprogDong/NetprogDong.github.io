---
layout: post
title:  "基于python语言的网络设备gRPC Telemetry简单实现"
date:   2020-02-18 10:35:36 +0530
description: python工具 网络监控
categories: python工具 网络监控
---

gRPC（Google Remote Procedure Call，Google远程过程调用）是Google发布的基于HTTP 2.0传输层协议承载的高性能开源软件框架，提供了支持多种编程语言的、对网络设备进行配置和管理的方法。通信双方可以基于该软件框架进行二次开发。  

### gRPC协议栈分层模型  

![图1](https://cdn.img.wenhairu.com/images/2020/02/18/mugNG.png "图1")

### gRPC Telemetry工作方式  
gRPC Telemetry有两种工作方式：  
* Dial-in模式：设备作为gRPC服务器，采集器作为gRPC客户端。由采集器主动向设备发起gRPC连接并订阅需要采集的数据信息。Dial-in模式适用于小规模网络和采集器需要向设备下发配置的场景。
* Dial-out模式：设备作为gRPC客户端，采集器作为gRPC服务器。设备主动和采集器建立gRPC连接，将设备上配置的订阅数据推送给采集器。Dial-out模式适用于网络设备较多的情况下向采集器提供设备数据信息。

后续所提及的均为Dial-out模式的实现。

### MACOS gRPC环境安装  
* 安装protoc工具(建议此步骤前先将python升级到2.7.10以上) 
  ```
  brew install protobuf
  ```
* 升级pip到最新版本
  ```
  sudo python -m pip install --upgrade pip
  ```
* 安装grpc的python模块
  ```
  sudo python -m pip install grpcio  
  或 
  sudo python -m pip install grpcio --ignore-installed
  
  sudo python -m pip install grpcio-tools
  ```

### H3C gRPC Telemetry简单实现  
1. 首先需要Dial-out对应的proto文件，这个文件可以直接让H3C的技术对接人员提供，也可以直接创建文件"grpc_dialout.proto"，并将如下代码复制到文件中。
   ```proto
   syntax = "proto2";
   package grpc_dialout;

   message DeviceInfo{
     required string producerName = 1;
     required string deviceName = 2;
     required string deviceModel = 3;
   }

   message DialoutMsg{
     required DeviceInfo deviceMsg = 1;
     required string sensorPath = 2;
     required string jsonData = 3;
   }

   message DialoutResponse{
     required string response = 1;
   }

   service GRPCDialout {
     rpc Dialout(stream DialoutMsg) returns (DialoutResponse);
   }
   ```
2. 使用protoc工具通过.proto文件生成相应的py文件（操作目录在.proto文件目录下）
   ```
   python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. grpc_dialout.proto
   ```
   执行如上命令后会看到同目录下会新生成2个文件："grpc_dialout_pb2.py"和"grpc_dialout_pb2_grpc.py"
   这两个文件就是开发要使用的grpc库文件。
3. 新建文件"h3c_grpc_dialout_test.py"，代码内容如下：
   ```python
   #! /usr/bin/env python
   # -*- coding: utf-8 -*-

   from concurrent import futures
   import time
   import grpc
   import h3c_grpc_dialout_pb2
   import h3c_grpc_dialout_pb2_grpc


   class GRPC_Dialout(h3c_grpc_dialout_pb2_grpc.GRPCDialoutServicer):
       def Dialout(self, request, context):
           for info in request:
               # 打印proto文件中定义的DeviceInfo消息
               print info.deviceMsg
               # 打印proto文件中定义的sensorPath消息
               print info.sensorPath
               # 打印指定sensorPath监控项的监控数据消息
               print info.jsonData
           return h3c_grpc_dialout_pb2.DialoutResponse(response='hello')


   def serve():
       # 这里通过threadpool来并发处理server的任务,最大并发数量10
       server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
       # 将对应的任务处理函数添加到rpc server中
       h3c_grpc_dialout_pb2_grpc.add_GRPCDialoutServicer_to_server(GRPC_Dialout(), server)
       # 这里使用的非安全接口,绑定tcp端口号44444,监听本地任意ip
       server.add_insecure_port('0.0.0.0:44444')
       server.start()
       try:
           while True:
               time.sleep(60*60*24)
       except KeyboardInterrupt:
           server.stop(0)


   if __name__ == '__main__':
       serve()
   ```
4. 运行文件
   python h3c_grpc_dialout_test.py
   此时服务端开启监听grpc进程，监听tcp端口44444
5. 交换机侧配置
   ```
   telemetry
    # 创建sensor-group test1
    sensor-group test1
     # 指定sensor-path
     sensor path ifmgr/statistics
    # 创建destination-group test1
    destination-group test1
     # 指定ip地址10.79.254.16，tcp端口44444
     ipv4-address 10.79.254.16 port 44444
    # 创建遥测任务subscription test1
    subscription test1
     # 调用sensor-group和destination-group
     sensor-group test1 sample-interval 5
     destination-group test1
   ```

### 效果图展示  
SNMP分钟级数据（绿线）与gRPC Telemetry秒级数据（黄线）对比图如下：

![图2](https://cdn.img.wenhairu.com/images/2020/02/18/muMnv.png "图2")




