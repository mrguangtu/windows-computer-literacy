# NAS外网访问的问题



很古早的时候，大概是200x年的时候，那个时侯IPV4还大行其道，个人宽带用户还可以获得公网IP。现在，肯定没有了。最近又开始折腾着玩儿，就发现内网NAS用原来的配置已经出不去了。



我这儿内网情况比较复杂。通过有电信猫+华为路由器。电信猫设置端口转发到华为路由器，华为路由器设置端口转发到nas。现在外网无法访问。如何测试，能发现是哪的端口转发问题吗？



于是搞了几段儿程序

```python
import socket

host = '106.38.171.75'  # 替换为您的实际公网IP
port = 80  # 替换为您转发到路由器的端口

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(5)
    try:
        s.connect((host, port))
        result = '电信猫的端口开放并可访问。'
    except Exception as e:
        result = f'电信猫的端口关闭或不可达: {e}'

print(result)
```

```python
import socket

router_ip = '192.168.1.5'  # 替换为您的华为路由器的局域网IP
port = 9926  # 替换为转发到NAS的端口

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(5)
    try:
        s.connect((router_ip, port))
        result = '华为路由器的端口开放并可访问。'
    except Exception as e:
        result = f'华为路由器的端口关闭或不可达: {e}'

print(result)
```

```python
import socket

nas_ip = '192.168.31.246'  # 替换为您的NAS局域网IP
nas_port = 9926  # 替换为NAS服务运行的端口

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(5)
    try:
        s.connect((nas_ip, nas_port))
        result = 'NAS服务端口开放并可访问。'
    except Exception as e:
        result = f'NAS服务端口关闭或不可达: {e}'

print(result)
```



检测下来，忽然恍然大悟。如今，电信已经没有IP了。不在电信的10.xx.xx.xx网段里面配置NAT，基本没可能了。



于是，能用的方法就只有下面这些了：



## **1. 使用内网穿透工具**

### **第三方服务**

- **ngrok**：这是一个非常流行的内网穿透工具，可以将您的本地服务通过一个公网地址暴露出来。您只需在NAS上安装ngrok客户端，并配置相应的端口映射。使用ngrok时，您会得到一个临时的公网URL，可以通过这个URL访问您的NAS。
- **frp**：这是另一个强大的内网穿透工具，可以设置一个公网服务器作为中转，转发请求到您的内网设备。您需要在一台具有公网IP的服务器上运行frp的服务端，并在NAS上运行frp的客户端。

## **2. 使用动态DNS（DDNS）**

虽然DDNS本身不能解决没有公网IP的问题，但如果您能够获得一个动态公网IP（例如通过ISP），则可以使用DDNS将其映射到一个固定域名上。这样，即使IP地址变化，您仍然可以通过域名访问。

## **3. 使用反向代理**

如果您有一个具备公网IP的云服务器，可以将其作为反向代理。配置反向代理后，所有来自公网的请求都会转发到您的内网NAS。这需要一定的技术知识来设置和维护。

## **4. 使用VPN**

搭建一个虚拟专用网络（VPN）也是一种解决方案。通过VPN，您可以将远程设备安全地连接到您的局域网，从而访问NAS。这需要在路由器或NAS上配置VPN服务器。

## **5. 使用花生壳等内网穿透软件**

花生壳是一款专门用于内网穿透的软件，无需公网IP即可实现远程访问。只需在NAS上安装花生壳客户端，进行简单配置，就可以生成一个外部访问地址。



这个时候，氪金的想法上来了，试了一试花生壳。结果发现，花生壳也做了策略，如果不给钱，干脆不让接入，真会赚钱。