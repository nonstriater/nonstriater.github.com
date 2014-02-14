---
layout: post
title: "DLNA协议使用"
description: ""
category: "基础"
tags: upnp dlna ios开发
---

CP(control point)端

## 发现设备

SSDP协议


### CP 主动搜索

当一个CP加入网络后，CP向多播地址*239.255.255.250：1900*发送查找search消息，满足查找条件的设备需要做出响应，响应信息中包含设备的一些基本信息（类型，标示符，指向设备详细描述信息的url）


设备查询通过HTTP协议扩展M-SEARCH方法实现（是内建在HTTPU/HTTPMU里）。

```ssdp
    
    M-SEARCH * HTTP/1.1
    HOST: 239.255.255.250:1900 
    MAN:"ssdp:discover" 
    MX:seconds to delay response 
    ST:search target

```


在设备接收到查询请求并且查询类型（ST字段值）与此设备匹配时，设备必须向源地址回应响应消息。最重要的信息是*LOCATION*

```http

    HTTP/1.1 200 OK
    CACHE-CONTROL:max-age = seconds until advertisement expires DATE:when response was generated
    EXT:
    LOCATION:URL for UPnP description for root device
￼   SERVER:OS/version UPnP/1.0 product/version
￼￼  ST:search target
    USN:advertisement UUID

```


SSDP使用UDP协议的1900端口传输。wireshark里过滤条件
```wireshark
    udp && http  &&  udp.dstport == 1900
```



### 设备宣告

当一个设备添加进网络后，设备可以向CP主动宣告自己的服务。多播*宣告消息*，CP监听标准端口来探测是否有感兴趣的服务加入网络。


宣告设备可用,设备的每个服务都会向多播地址发送*宣告消息*  ssdp:alive，每个宣告消息包含4个主要消息：

```ssdp
    NT
    USN
    LOCATION
    CACHE-CONTROL
```


设备查询通过HTTP协议扩展NOTIFY方法实现，基于UDP传输。

```ssdp

￼NOTIFY * HTTP/1.1
￼HOST: 239.255.255.250:1900
￼￼
￼LOCATION:URL for UPnP description for root device
￼NT: urn:schemas-upnp-org:service:ConnectionManager:1\r\n
￼NTS:ssdp:alive
￼Server: Linux/2.6.31.3, UPnP/1.0, Portable SDK for UPnP devices/1.6.13
￼USN:advertisement UUID

```



宣告设备不可用 ssdp:byebye

```ssdp

￼NOTIFY * HTTP/1.1
￼HOST: 239.255.255.250:1900
￼NT: urn:schemas-upnp-org:service:ConnectionManager:1\r\n
￼NTS:ssdp:byebye
￼￼USN:advertisement UUID

```


HOST:
NT： 通知类型
NTS：通知子类型，必须是 ssdp:alive or ssdp:byebye
USN: 唯一服务名称 如：USN: uuid:xx-02d1050199c4_MR::urn:schemas-upnp-org:service:ConnectionManager:1\r\n
LOCATION
SERVER


## 获得设备描述信息

通过上面得到的URL，使用 HTTP get方法就可以获得设备描述信息。

xml文件

设备信息，
服务信息，
控制、事件触发、展示的url
状态变量

```xml
<?xml version="1.0"?>
<root xmlns="urn:schemas-upnp-org:device-1-0">
    <specVersion>
        <major>1</major>
￼￼￼
        <minor>0</minor>
    </specVersion>
    <URLBase>base URL for all relative URLs</URLBase>
    <device>
        <deviceType>urn:schemas-upnp-org:device:deviceType:v</deviceType>
        <friendlyName>short user-friendly title</friendlyName>
        <manufacturer>manufacturer name</manufacturer>
        <manufacturerURL>URL to manufacturer site</manufacturerURL>
        <modelDescription>long user-friendly title</modelDescription>
        <modelName>model name</modelName>
        <modelNumber>model number</modelNumber>
        <modelURL>URL to model site</modelURL>
        <serialNumber>manufacturer's serial number</serialNumber>
        <UDN>uuid:UUID</UDN>
        <UPC>Universal Product Code</UPC>
        <presentationURL>URL for presentation</presentationURL>

        <iconList>
            <icon>
                <mimetype>image/format</mimetype>
                <width>horizontal pixels</width>
￼￼
                <height>vertical pixels</height>
                <depth>color depth</depth>
                <url>URL to icon</url>
            </icon>
            XML to declare other icons, if any, go here
￼       </iconList>
￼￼
        <serviceList>
￼￼          <service>
￼￼              <serviceType>urn:schemas-upnp-org:service:serviceType:v</serviceType>
￼￼￼￼￼￼￼         <serviceId>urn:upnp-org:serviceId:serviceID</serviceId>
￼￼￼￼￼￼          <SCPDURL>URL to service description</SCPDURL>
                ￼<controlURL>URL for control</controlURL>
￼￼￼              <eventSubURL>URL for eventing</eventSubURL>
￼￼￼         </service>
            <service> ... </service>
￼       </serviceList>

￼￼      <deviceList> 
            Description of embedded devices added by UPnP vendor (if any) go here
        ￼</deviceList>
￼￼   
￼￼￼ </device>
￼</root>

```

UDN  : 设备uuid
SCPDURL:   关于服务详细描述
controlURL: control url
eventSubURL:



## 控制设备

SOAP协议

根据设备提供的控制服务的URL，CP可以按照指定的控制消息格式向设备发送消息，来控制设备。

发出动作实质上是一种远程过程调用（RPC）。控制点可以向设备发出动作，也可以轮询状态变量值。

UPnp 使用 Soap来向设备提供控制消息，并将结果或错误返回控制点。比如对于一款音箱设备，我们可能需要的控制动作有：播放（播放当前歌曲，播放指定歌曲），暂停，音量调节，上一首，下一首

###  发起动作 

CP 采用以下格式的POST方法发送一个请求：

```soap

    POST  path of control URL
    CONTENT-LENGTH: bytes in body
    CONTENT-TYPE: text/xml; charset="utf-8"
    SOAPACTION: "urn:schemas-upnp-org:service:serviceType:v#actionNa

    <s:Envelope 
    xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" 
    s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
     ￼<s:Body>
￼￼￼     <u:actionName xmlns:u="urn:schemas-upnp-org:service:serviceType:v">
￼￼￼￼￼       <argumentName>in arg value</argumentName>
             other in args and their values go here, if any
        </u:actionName>
￼￼￼   </s:Body>
    </s:Envelope>


```


响应

```soap

    M-POST path of control URL HTTP/1.1 
    HOST: host of control URL:port of control URL 
    CONTENT-LENGTH: bytes in body 
    CONTENT-TYPE: text/xml; charset="utf-8"
￼￼￼
￼   MAN: "http://schemas.xmlsoap.org/soap/envelope/"; ns=01
￼￼  01-SOAPACTION: "urn:schemas-upnp-org:service:serviceType:v#actionName"
```

### 查询变量



## 事件订阅

GENA格式（通用事件通知架构）

CP可以订阅设备的一些状态变量，当设备状态变量变化时，设备发出订阅事件消息，消息包括订阅变量名和当前值。订阅者都会收到全部的事件消息。


```gena

NOTIFY delivery path HTTP/1.1
HOST: delivery host:delivery port
CONTENT-TYPE: text/xml
NT: upnp:event
NTS: upnp:propchange
SID: uuid:subscription-UUID
SEQ: event key
<e:propertyset xmlns:e="urn:schemas-upnp-org:event-1-0">
 <e:property>
 <variableName>new value</variableName>
 </e:property>
  Other variable names and values (if any) go here
</e:propertyset>

```



##  AV 设备架构

AV 设备架构的3个组件：
任何一个设备都是1~3个组件的组合。我们的智能手机就是典型的3各组件的组合。

Media Server ： 保存多媒体资料。提供内容目录管理(ContentDirectory),连接管理（ConnectionManager）,音视频传输(AVTransport)，前2个必须提供。
Media Render ： 获取多媒体数据，并播放渲染。提供的服务是播放控制(RenderingControl),连接管理(ConnectionManager) , 音视频传输（AVTransport），前2个必须提供。
Control Point： 控制端

一个设备在做不同的动作时，扮演的角色也可能跟着变化。如手机向智能音响推送一首歌曲时，手机扮演的是CP和MediaServer的角色；手机暂停音响的播放时，扮演的是纯CP的角色。


serviceType
urn:schemas-upnp-org:service:ConnectionManager:1
urn:schemas-upnp-org:service:RenderingControl:1
urn:schemas-upnp-org:service:AVTransport:1
urn:schemas-upnp-org:service:ContentDirectory:1

http://192.168.4.102:49154/renderer.xml
http://192.168.4.102:49154/mediaserver.xml


### 内容管理 ContentDirectory

```xml

<DIDL-Lite xmlns="urn:schemas-upnp-org:metadata-1-0/DIDL-Lite/" 
            xmlns:dc="http://purl.org/dc/elements/1.1/" 
            xmlns:upnp="urn:schemas-upnp-org:metadata-1-0/upnp/">
    <container id="1" parentID="0" childCount="2" restricted="true">
    <upnp:class>object.container.storageFolder</upnp:class>
    <dc:title>sdcard</dc:title>
    </container>
</DIDL-Lite>

```








