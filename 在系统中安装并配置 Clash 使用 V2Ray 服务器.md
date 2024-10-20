# 完整指南：在系统中安装并配置 Clash 使用 V2Ray 服务器

#clash #v2ray

本指南旨在提供从零开始在您的系统中安装 Clash 客户端并配置一个CentOS的 V2Ray 代理服务器的详细步骤。这里假设您正在使用的服务器是 Unix-like 系统（如 macOS 或 Linux）。客户端是Windows 等，用户将需要下载适用于对应系统的 Clash 客户端并进行相应的调整。

## 第一步：安装 V2Ray

1. **连接服务器**：使用 SSH 客户端连接到你的 CentOS 服务器。
2. **修改安全组：** 购买服务器后，注意修改安全组。建议把所有udp和tcp都配通。防止一切正常但无法访问服务器。
3. **下载安装脚本**：获取 V2Ray 的安装脚本，可以使用 `curl` 或 `wget` 工具。例如：

   ~~~shell
   sudo yum install -y curl
   curl -O https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh
   curl -O https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-dat-release.sh
   ~~~

3. **执行安装脚本**：赋予脚本执行权限，并运行它们来安装 V2Ray。

```shell
   sudo bash install-release.sh
   sudo bash install-dat-release.sh
```


## 第二步：配置 V2Ray

1. **编辑配置文件**：V2Ray 的配置文件通常位于 `/usr/local/etc/v2ray/config.json`。使用文本编辑器打开并编辑它。

```shell
   sudo vim /usr/local/etc/v2ray/config.json
```

2. **配置 VMess**：以下是一个简单的 VMess 配置示例。你需要将 `"your_uuid"` 替换为你自己的 UUID。
生成uuid可以使用：uuidgen
```json
   {
     "inbounds": [{
       "port": 12345,
       "protocol": "vmess",
       "settings": {
         "clients": [
           {
             "id": "00dbd5b6-298e-4728-90ec-27ceea3a887e",
             "alterId": 0
           }
         ]
       }
     }],
     "outbounds": [{
       "protocol": "freedom",
       "settings": {}
     }]
   }
```

## 第三步：启动 V2Ray

```shell
/usr/local/bin/v2ray run -c /usr/local/etc/v2ray/config.json
```

## 第四步：设置 Systemd 服务

1. **修改 Systemd 服务配置文件**：如果不打算更新 `systemd`，你可以尝试编辑 `v2ray.service` 文件，移除可能导致问题的配置项。

   ```shell
   sudo vi /etc/systemd/system/v2ray.service
   ```

   `v2ray.service` 文件的内容：

   ```ini
   [Unit]
   Description=V2Ray Service
   After=network.target

   [Service]
   Type=simple
   User=nobody
   ExecStart=/usr/local/bin/v2ray run -c /usr/local/etc/v2ray/config.json
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```

2. **重新加载 Systemd 配置**：

   ````shell
   sudo systemctl daemon-reload
   
3. **启动 V2Ray 服务**：

   ````shell
   sudo systemctl start v2ray
   
4. **设置 V2Ray 开机自启**：

   ````shell
   sudo systemctl enable v2ray
   
5. **检查 V2Ray 状态**：

   ````shell
   sudo systemctl status v2ray
   
6. **停止服务**：

   ````shell
   sudo systemctl stop v2ray
   
7. **查看日志**：

   ````shell
   sudo journalctl -u v2ray

## 第五步：生成 UUID

在 Linux 命令行中：

```shell
uuidgen
```

这将输出一个新的 UUID，例如：`123e4567-e89b-12d3-a456-426614174000`。将此 UUID 用于 V2Ray 配置。

## 第六步：客户端配置

在你的客户端设备上，安装 V2Ray 客户端软件，并配置服务器信息。

Clash 客户端配置示例：

```yaml
# Profile Template for clash

proxies:
  - name: "MyV2RayServer"
    type: vmess
    server: your_server_ip # 你的 V2Ray 服务器地址
    port: your_server_port # 你的 V2Ray 服务器端口
    uuid: your_uuid # 你的 V2Ray UUID
    alterId: 0
    cipher: auto

proxy-groups:
- name: Proxy
  type: select
  proxies:
    - MyV2RayServer

rules:
  - MATCH,Proxy
```

一个更加复杂的示例：
```yaml
# Profile Template for clash verge
allow-lan: true
bind-address: '*'
mode: rule
log-level: info
external-controller: '127.0.0.1:9090'
dns:
    enable: false
    ipv6: false
    default-nameserver: [223.5.5.5, 119.29.29.29]
    enhanced-mode: fake-ip
    fake-ip-range: 198.18.0.1/16
    use-hosts: true
    nameserver: ['https://doh.pub/dns-query', 'https://dns.alidns.com/dns-query']
    fallback: ['https://doh.dns.sb/dns-query', 'https://dns.cloudflare.com/dns-query', 'https://dns.twnic.tw/dns-query', 'tls://8.8.4.4:853']
    fallback-filter: { geoip: true, ipcidr: [240.0.0.0/4, 0.0.0.0/32] }

proxies:
  - { name: MyV2RayServer1, type: vmess, server: 23.105.201.244, port: 12345, uuid: c99a66c1-dca1-4f09-942b-deb8a7828793, alterId: 0, cipher: auto }
  - { name: MyV2RayServer2, type: vmess, server: 45.41.8.61, port: 12345, uuid: 00dbd5b6-298e-4728-90ec-27ceea3a887e, alterId: 0, cipher: auto }


proxy-groups:
  - { name: MyProxy, type: select, proxies: [自动选择, 故障转移, 'MyV2RayServer1', 'MyV2RayServer2'] }
  - { name: 自动选择, type: url-test, proxies: ['MyV2RayServer1', 'MyV2RayServer2'], url: 'http://www.gstatic.com/generate_204', interval: 86400 }
  - { name: 故障转移, type: fallback, proxies: ['MyV2RayServer1', 'MyV2RayServer2'], url: 'http://www.gstatic.com/generate_204', interval: 7200 }


rules:
   - 'DOMAIN,www.yiyuanjichang.net,DIRECT'
   - 'DOMAIN-SUFFIX,xn--ngstr-lra8j.com,MyProxy'
   - 'DOMAIN-SUFFIX,services.googleapis.cn,MyProxy'
   - 'DOMAIN,safebrowsing.urlsec.qq.com,DIRECT'
   - 'DOMAIN,safebrowsing.googleapis.com,DIRECT'
   - 'DOMAIN,developer.apple.com,MyProxy'
   - 'DOMAIN-SUFFIX,digicert.com,MyProxy'
   - 'DOMAIN-SUFFIX,bilibili.com,DIRECT'
   - 'DOMAIN,ocsp.apple.com,MyProxy'
   - 'DOMAIN,ocsp.comodoca.com,MyProxy'
   - 'DOMAIN,ocsp.usertrust.com,MyProxy'
   - 'DOMAIN,ocsp.sectigo.com,MyProxy'
   - 'DOMAIN,ocsp.verisign.net,MyProxy'
   - 'DOMAIN-SUFFIX,apple-dns.net,MyProxy'
   - 'DOMAIN,testflight.apple.com,MyProxy'
   - 'DOMAIN,sandbox.itunes.apple.com,MyProxy'
   - 'DOMAIN,itunes.apple.com,MyProxy'
   - 'DOMAIN-SUFFIX,apps.apple.com,MyProxy'
   - 'DOMAIN-SUFFIX,blobstore.apple.com,MyProxy'
   - 'DOMAIN,cvws.icloud-content.com,MyProxy'
   - 'DOMAIN-SUFFIX,mzstatic.com,DIRECT'
   - 'DOMAIN-SUFFIX,itunes.apple.com,DIRECT'
   - 'DOMAIN-SUFFIX,icloud.com,DIRECT'
   - 'DOMAIN-SUFFIX,icloud-content.com,DIRECT'
   - 'DOMAIN-SUFFIX,me.com,DIRECT'
   - 'DOMAIN-SUFFIX,aaplimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,cdn20.com,DIRECT'
   - 'DOMAIN-SUFFIX,cdn-apple.com,DIRECT'
   - 'DOMAIN-SUFFIX,akadns.net,DIRECT'
   - 'DOMAIN-SUFFIX,akamaiedge.net,DIRECT'
   - 'DOMAIN-SUFFIX,edgekey.net,DIRECT'
   - 'DOMAIN-SUFFIX,mwcloudcdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,mwcname.com,DIRECT'
   - 'DOMAIN-SUFFIX,apple.com,DIRECT'
   - 'DOMAIN-SUFFIX,apple-cloudkit.com,DIRECT'
   - 'DOMAIN-SUFFIX,apple-mapkit.com,DIRECT'
   - 'DOMAIN-SUFFIX,126.com,DIRECT'
   - 'DOMAIN-SUFFIX,126.net,DIRECT'
   - 'DOMAIN-SUFFIX,127.net,DIRECT'
   - 'DOMAIN-SUFFIX,163.com,DIRECT'
   - 'DOMAIN-SUFFIX,360buyimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,36kr.com,DIRECT'
   - 'DOMAIN-SUFFIX,acfun.tv,DIRECT'
   - 'DOMAIN-SUFFIX,air-matters.com,DIRECT'
   - 'DOMAIN-SUFFIX,aixifan.com,DIRECT'
   - 'DOMAIN-KEYWORD,alicdn,DIRECT'
   - 'DOMAIN-KEYWORD,alipay,DIRECT'
   - 'DOMAIN-KEYWORD,taobao,DIRECT'
   - 'DOMAIN-SUFFIX,amap.com,DIRECT'
   - 'DOMAIN-SUFFIX,autonavi.com,DIRECT'
   - 'DOMAIN-KEYWORD,baidu,DIRECT'
   - 'DOMAIN-SUFFIX,bdimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,bdstatic.com,DIRECT'
   - 'DOMAIN-SUFFIX,bilibili.com,DIRECT'
   - 'DOMAIN-SUFFIX,bilivideo.com,DIRECT'
   - 'DOMAIN-SUFFIX,caiyunapp.com,DIRECT'
   - 'DOMAIN-SUFFIX,clouddn.com,DIRECT'
   - 'DOMAIN-SUFFIX,cnbeta.com,DIRECT'
   - 'DOMAIN-SUFFIX,cnbetacdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,cootekservice.com,DIRECT'
   - 'DOMAIN-SUFFIX,csdn.net,DIRECT'
   - 'DOMAIN-SUFFIX,ctrip.com,DIRECT'
   - 'DOMAIN-SUFFIX,dgtle.com,DIRECT'
   - 'DOMAIN-SUFFIX,dianping.com,DIRECT'
   - 'DOMAIN-SUFFIX,douban.com,DIRECT'
   - 'DOMAIN-SUFFIX,doubanio.com,DIRECT'
   - 'DOMAIN-SUFFIX,duokan.com,DIRECT'
   - 'DOMAIN-SUFFIX,easou.com,DIRECT'
   - 'DOMAIN-SUFFIX,ele.me,DIRECT'
   - 'DOMAIN-SUFFIX,feng.com,DIRECT'
   - 'DOMAIN-SUFFIX,fir.im,DIRECT'
   - 'DOMAIN-SUFFIX,frdic.com,DIRECT'
   - 'DOMAIN-SUFFIX,g-cores.com,DIRECT'
   - 'DOMAIN-SUFFIX,godic.net,DIRECT'
   - 'DOMAIN-SUFFIX,gtimg.com,DIRECT'
   - 'DOMAIN,cdn.hockeyapp.net,DIRECT'
   - 'DOMAIN-SUFFIX,hongxiu.com,DIRECT'
   - 'DOMAIN-SUFFIX,hxcdn.net,DIRECT'
   - 'DOMAIN-SUFFIX,iciba.com,DIRECT'
   - 'DOMAIN-SUFFIX,ifeng.com,DIRECT'
   - 'DOMAIN-SUFFIX,ifengimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,ipip.net,DIRECT'
   - 'DOMAIN-SUFFIX,iqiyi.com,DIRECT'
   - 'DOMAIN-SUFFIX,jd.com,DIRECT'
   - 'DOMAIN-SUFFIX,jianshu.com,DIRECT'
   - 'DOMAIN-SUFFIX,knewone.com,DIRECT'
   - 'DOMAIN-SUFFIX,le.com,DIRECT'
   - 'DOMAIN-SUFFIX,lecloud.com,DIRECT'
   - 'DOMAIN-SUFFIX,lemicp.com,DIRECT'
   - 'DOMAIN-SUFFIX,licdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,luoo.net,DIRECT'
   - 'DOMAIN-SUFFIX,meituan.com,DIRECT'
   - 'DOMAIN-SUFFIX,meituan.net,DIRECT'
   - 'DOMAIN-SUFFIX,mi.com,DIRECT'
   - 'DOMAIN-SUFFIX,miaopai.com,DIRECT'
   - 'DOMAIN-SUFFIX,microsoft.com,DIRECT'
   - 'DOMAIN-SUFFIX,microsoftonline.com,DIRECT'
   - 'DOMAIN-SUFFIX,miui.com,DIRECT'
   - 'DOMAIN-SUFFIX,miwifi.com,DIRECT'
   - 'DOMAIN-SUFFIX,mob.com,DIRECT'
   - 'DOMAIN-SUFFIX,netease.com,DIRECT'
   - 'DOMAIN-SUFFIX,office.com,DIRECT'
   - 'DOMAIN-SUFFIX,office365.com,DIRECT'
   - 'DOMAIN-KEYWORD,officecdn,DIRECT'
   - 'DOMAIN-SUFFIX,oschina.net,DIRECT'
   - 'DOMAIN-SUFFIX,ppsimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,pstatp.com,DIRECT'
   - 'DOMAIN-SUFFIX,qcloud.com,DIRECT'
   - 'DOMAIN-SUFFIX,qdaily.com,DIRECT'
   - 'DOMAIN-SUFFIX,qdmm.com,DIRECT'
   - 'DOMAIN-SUFFIX,qhimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,qhres.com,DIRECT'
   - 'DOMAIN-SUFFIX,qidian.com,DIRECT'
   - 'DOMAIN-SUFFIX,qihucdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,qiniu.com,DIRECT'
   - 'DOMAIN-SUFFIX,qiniucdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,qiyipic.com,DIRECT'
   - 'DOMAIN-SUFFIX,qq.com,DIRECT'
   - 'DOMAIN-SUFFIX,qqurl.com,DIRECT'
   - 'DOMAIN-SUFFIX,rarbg.to,DIRECT'
   - 'DOMAIN-SUFFIX,ruguoapp.com,DIRECT'
   - 'DOMAIN-SUFFIX,segmentfault.com,DIRECT'
   - 'DOMAIN-SUFFIX,sinaapp.com,DIRECT'
   - 'DOMAIN-SUFFIX,smzdm.com,DIRECT'
   - 'DOMAIN-SUFFIX,snapdrop.net,DIRECT'
   - 'DOMAIN-SUFFIX,sogou.com,DIRECT'
   - 'DOMAIN-SUFFIX,sogoucdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,sohu.com,DIRECT'
   - 'DOMAIN-SUFFIX,soku.com,DIRECT'
   - 'DOMAIN-SUFFIX,speedtest.net,DIRECT'
   - 'DOMAIN-SUFFIX,sspai.com,DIRECT'
   - 'DOMAIN-SUFFIX,suning.com,DIRECT'
   - 'DOMAIN-SUFFIX,taobao.com,DIRECT'
   - 'DOMAIN-SUFFIX,tencent.com,DIRECT'
   - 'DOMAIN-SUFFIX,tenpay.com,DIRECT'
   - 'DOMAIN-SUFFIX,tianyancha.com,DIRECT'
   - 'DOMAIN-SUFFIX,tmall.com,DIRECT'
   - 'DOMAIN-SUFFIX,tudou.com,DIRECT'
   - 'DOMAIN-SUFFIX,umetrip.com,DIRECT'
   - 'DOMAIN-SUFFIX,upaiyun.com,DIRECT'
   - 'DOMAIN-SUFFIX,upyun.com,DIRECT'
   - 'DOMAIN-SUFFIX,veryzhun.com,DIRECT'
   - 'DOMAIN-SUFFIX,weather.com,DIRECT'
   - 'DOMAIN-SUFFIX,weibo.com,DIRECT'
   - 'DOMAIN-SUFFIX,xiami.com,DIRECT'
   - 'DOMAIN-SUFFIX,xiami.net,DIRECT'
   - 'DOMAIN-SUFFIX,xiaomicp.com,DIRECT'
   - 'DOMAIN-SUFFIX,ximalaya.com,DIRECT'
   - 'DOMAIN-SUFFIX,xmcdn.com,DIRECT'
   - 'DOMAIN-SUFFIX,xunlei.com,DIRECT'
   - 'DOMAIN-SUFFIX,yhd.com,DIRECT'
   - 'DOMAIN-SUFFIX,yihaodianimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,yinxiang.com,DIRECT'
   - 'DOMAIN-SUFFIX,ykimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,youdao.com,DIRECT'
   - 'DOMAIN-SUFFIX,youku.com,DIRECT'
   - 'DOMAIN-SUFFIX,zealer.com,DIRECT'
   - 'DOMAIN-SUFFIX,zhihu.com,DIRECT'
   - 'DOMAIN-SUFFIX,zhimg.com,DIRECT'
   - 'DOMAIN-SUFFIX,zimuzu.tv,DIRECT'
   - 'DOMAIN-SUFFIX,zoho.com,DIRECT'
   - 'DOMAIN-KEYWORD,amazon,MyProxy'
   - 'DOMAIN-KEYWORD,google,MyProxy'
   - 'DOMAIN-KEYWORD,gmail,MyProxy'
   - 'DOMAIN-KEYWORD,youtube,MyProxy'
   - 'DOMAIN-KEYWORD,facebook,MyProxy'
   - 'DOMAIN-SUFFIX,fb.me,MyProxy'
   - 'DOMAIN-SUFFIX,fbcdn.net,MyProxy'
   - 'DOMAIN-KEYWORD,twitter,MyProxy'
   - 'DOMAIN-KEYWORD,instagram,MyProxy'
   - 'DOMAIN-KEYWORD,dropbox,MyProxy'
   - 'DOMAIN-SUFFIX,twimg.com,MyProxy'
   - 'DOMAIN-KEYWORD,blogspot,MyProxy'
   - 'DOMAIN-SUFFIX,youtu.be,MyProxy'
   - 'DOMAIN-KEYWORD,whatsapp,MyProxy'
   - 'DOMAIN-KEYWORD,admarvel,REJECT'
   - 'DOMAIN-KEYWORD,admaster,REJECT'
   - 'DOMAIN-KEYWORD,adsage,REJECT'
   - 'DOMAIN-KEYWORD,adsmogo,REJECT'
   - 'DOMAIN-KEYWORD,adsrvmedia,REJECT'
   - 'DOMAIN-KEYWORD,adwords,REJECT'
   - 'DOMAIN-KEYWORD,adservice,REJECT'
   - 'DOMAIN-SUFFIX,appsflyer.com,REJECT'
   - 'DOMAIN-KEYWORD,domob,REJECT'
   - 'DOMAIN-SUFFIX,doubleclick.net,REJECT'
   - 'DOMAIN-KEYWORD,duomeng,REJECT'
   - 'DOMAIN-KEYWORD,dwtrack,REJECT'
   - 'DOMAIN-KEYWORD,guanggao,REJECT'
   - 'DOMAIN-KEYWORD,lianmeng,REJECT'
   - 'DOMAIN-SUFFIX,mmstat.com,REJECT'
   - 'DOMAIN-KEYWORD,mopub,REJECT'
   - 'DOMAIN-KEYWORD,omgmta,REJECT'
   - 'DOMAIN-KEYWORD,openx,REJECT'
   - 'DOMAIN-KEYWORD,partnerad,REJECT'
   - 'DOMAIN-KEYWORD,pingfore,REJECT'
   - 'DOMAIN-KEYWORD,supersonicads,REJECT'
   - 'DOMAIN-KEYWORD,uedas,REJECT'
   - 'DOMAIN-KEYWORD,umeng,REJECT'
   - 'DOMAIN-KEYWORD,usage,REJECT'
   - 'DOMAIN-SUFFIX,vungle.com,REJECT'
   - 'DOMAIN-KEYWORD,wlmonitor,REJECT'
   - 'DOMAIN-KEYWORD,zjtoolbar,REJECT'
   - 'DOMAIN-SUFFIX,9to5mac.com,MyProxy'
   - 'DOMAIN-SUFFIX,abpchina.org,MyProxy'
   - 'DOMAIN-SUFFIX,adblockplus.org,MyProxy'
   - 'DOMAIN-SUFFIX,adobe.com,MyProxy'
   - 'DOMAIN-SUFFIX,akamaized.net,MyProxy'
   - 'DOMAIN-SUFFIX,alfredapp.com,MyProxy'
   - 'DOMAIN-SUFFIX,amplitude.com,MyProxy'
   - 'DOMAIN-SUFFIX,ampproject.org,MyProxy'
   - 'DOMAIN-SUFFIX,android.com,MyProxy'
   - 'DOMAIN-SUFFIX,angularjs.org,MyProxy'
   - 'DOMAIN-SUFFIX,aolcdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,apkpure.com,MyProxy'
   - 'DOMAIN-SUFFIX,appledaily.com,MyProxy'
   - 'DOMAIN-SUFFIX,appshopper.com,MyProxy'
   - 'DOMAIN-SUFFIX,appspot.com,MyProxy'
   - 'DOMAIN-SUFFIX,arcgis.com,MyProxy'
   - 'DOMAIN-SUFFIX,archive.org,MyProxy'
   - 'DOMAIN-SUFFIX,armorgames.com,MyProxy'
   - 'DOMAIN-SUFFIX,aspnetcdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,att.com,MyProxy'
   - 'DOMAIN-SUFFIX,awsstatic.com,MyProxy'
   - 'DOMAIN-SUFFIX,azureedge.net,MyProxy'
   - 'DOMAIN-SUFFIX,azurewebsites.net,MyProxy'
   - 'DOMAIN-SUFFIX,bing.com,MyProxy'
   - 'DOMAIN-SUFFIX,bintray.com,MyProxy'
   - 'DOMAIN-SUFFIX,bit.com,MyProxy'
   - 'DOMAIN-SUFFIX,bit.ly,MyProxy'
   - 'DOMAIN-SUFFIX,bitbucket.org,MyProxy'
   - 'DOMAIN-SUFFIX,bjango.com,MyProxy'
   - 'DOMAIN-SUFFIX,bkrtx.com,MyProxy'
   - 'DOMAIN-SUFFIX,blog.com,MyProxy'
   - 'DOMAIN-SUFFIX,blogcdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,blogger.com,MyProxy'
   - 'DOMAIN-SUFFIX,blogsmithmedia.com,MyProxy'
   - 'DOMAIN-SUFFIX,blogspot.com,MyProxy'
   - 'DOMAIN-SUFFIX,blogspot.hk,MyProxy'
   - 'DOMAIN-SUFFIX,bloomberg.com,MyProxy'
   - 'DOMAIN-SUFFIX,box.com,MyProxy'
   - 'DOMAIN-SUFFIX,box.net,MyProxy'
   - 'DOMAIN-SUFFIX,cachefly.net,MyProxy'
   - 'DOMAIN-SUFFIX,chromium.org,MyProxy'
   - 'DOMAIN-SUFFIX,cl.ly,MyProxy'
   - 'DOMAIN-SUFFIX,cloudflare.com,MyProxy'
   - 'DOMAIN-SUFFIX,cloudfront.net,MyProxy'
   - 'DOMAIN-SUFFIX,cloudmagic.com,MyProxy'
   - 'DOMAIN-SUFFIX,cmail19.com,MyProxy'
   - 'DOMAIN-SUFFIX,cnet.com,MyProxy'
   - 'DOMAIN-SUFFIX,cocoapods.org,MyProxy'
   - 'DOMAIN-SUFFIX,comodoca.com,MyProxy'
   - 'DOMAIN-SUFFIX,crashlytics.com,MyProxy'
   - 'DOMAIN-SUFFIX,culturedcode.com,MyProxy'
   - 'DOMAIN-SUFFIX,d.pr,MyProxy'
   - 'DOMAIN-SUFFIX,danilo.to,MyProxy'
   - 'DOMAIN-SUFFIX,dayone.me,MyProxy'
   - 'DOMAIN-SUFFIX,db.tt,MyProxy'
   - 'DOMAIN-SUFFIX,deskconnect.com,MyProxy'
   - 'DOMAIN-SUFFIX,disq.us,MyProxy'
   - 'DOMAIN-SUFFIX,disqus.com,MyProxy'
   - 'DOMAIN-SUFFIX,disquscdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,dnsimple.com,MyProxy'
   - 'DOMAIN-SUFFIX,docker.com,MyProxy'
   - 'DOMAIN-SUFFIX,dribbble.com,MyProxy'
   - 'DOMAIN-SUFFIX,droplr.com,MyProxy'
   - 'DOMAIN-SUFFIX,duckduckgo.com,MyProxy'
   - 'DOMAIN-SUFFIX,dueapp.com,MyProxy'
   - 'DOMAIN-SUFFIX,dytt8.net,MyProxy'
   - 'DOMAIN-SUFFIX,edgecastcdn.net,MyProxy'
   - 'DOMAIN-SUFFIX,edgekey.net,MyProxy'
   - 'DOMAIN-SUFFIX,edgesuite.net,MyProxy'
   - 'DOMAIN-SUFFIX,engadget.com,MyProxy'
   - 'DOMAIN-SUFFIX,entrust.net,MyProxy'
   - 'DOMAIN-SUFFIX,eurekavpt.com,MyProxy'
   - 'DOMAIN-SUFFIX,evernote.com,MyProxy'
   - 'DOMAIN-SUFFIX,fabric.io,MyProxy'
   - 'DOMAIN-SUFFIX,fast.com,MyProxy'
   - 'DOMAIN-SUFFIX,fastly.net,MyProxy'
   - 'DOMAIN-SUFFIX,fc2.com,MyProxy'
   - 'DOMAIN-SUFFIX,feedburner.com,MyProxy'
   - 'DOMAIN-SUFFIX,feedly.com,MyProxy'
   - 'DOMAIN-SUFFIX,feedsportal.com,MyProxy'
   - 'DOMAIN-SUFFIX,fiftythree.com,MyProxy'
   - 'DOMAIN-SUFFIX,firebaseio.com,MyProxy'
   - 'DOMAIN-SUFFIX,flexibits.com,MyProxy'
   - 'DOMAIN-SUFFIX,flickr.com,MyProxy'
   - 'DOMAIN-SUFFIX,flipboard.com,MyProxy'
   - 'DOMAIN-SUFFIX,g.co,MyProxy'
   - 'DOMAIN-SUFFIX,gabia.net,MyProxy'
   - 'DOMAIN-SUFFIX,geni.us,MyProxy'
   - 'DOMAIN-SUFFIX,gfx.ms,MyProxy'
   - 'DOMAIN-SUFFIX,ggpht.com,MyProxy'
   - 'DOMAIN-SUFFIX,ghostnoteapp.com,MyProxy'
   - 'DOMAIN-SUFFIX,git.io,MyProxy'
   - 'DOMAIN-KEYWORD,github,MyProxy'
   - 'DOMAIN-SUFFIX,globalsign.com,MyProxy'
   - 'DOMAIN-SUFFIX,gmodules.com,MyProxy'
   - 'DOMAIN-SUFFIX,godaddy.com,MyProxy'
   - 'DOMAIN-SUFFIX,golang.org,MyProxy'
   - 'DOMAIN-SUFFIX,gongm.in,MyProxy'
   - 'DOMAIN-SUFFIX,goo.gl,MyProxy'
   - 'DOMAIN-SUFFIX,goodreaders.com,MyProxy'
   - 'DOMAIN-SUFFIX,goodreads.com,MyProxy'
   - 'DOMAIN-SUFFIX,gravatar.com,MyProxy'
   - 'DOMAIN-SUFFIX,gstatic.com,MyProxy'
   - 'DOMAIN-SUFFIX,gvt0.com,MyProxy'
   - 'DOMAIN-SUFFIX,hockeyapp.net,MyProxy'
   - 'DOMAIN-SUFFIX,hotmail.com,MyProxy'
   - 'DOMAIN-SUFFIX,icons8.com,MyProxy'
   - 'DOMAIN-SUFFIX,ifixit.com,MyProxy'
   - 'DOMAIN-SUFFIX,ift.tt,MyProxy'
   - 'DOMAIN-SUFFIX,ifttt.com,MyProxy'
   - 'DOMAIN-SUFFIX,iherb.com,MyProxy'
   - 'DOMAIN-SUFFIX,imageshack.us,MyProxy'
   - 'DOMAIN-SUFFIX,img.ly,MyProxy'
   - 'DOMAIN-SUFFIX,imgur.com,MyProxy'
   - 'DOMAIN-SUFFIX,imore.com,MyProxy'
   - 'DOMAIN-SUFFIX,instapaper.com,MyProxy'
   - 'DOMAIN-SUFFIX,ipn.li,MyProxy'
   - 'DOMAIN-SUFFIX,is.gd,MyProxy'
   - 'DOMAIN-SUFFIX,issuu.com,MyProxy'
   - 'DOMAIN-SUFFIX,itgonglun.com,MyProxy'
   - 'DOMAIN-SUFFIX,itun.es,MyProxy'
   - 'DOMAIN-SUFFIX,ixquick.com,MyProxy'
   - 'DOMAIN-SUFFIX,j.mp,MyProxy'
   - 'DOMAIN-SUFFIX,js.revsci.net,MyProxy'
   - 'DOMAIN-SUFFIX,jshint.com,MyProxy'
   - 'DOMAIN-SUFFIX,jtvnw.net,MyProxy'
   - 'DOMAIN-SUFFIX,justgetflux.com,MyProxy'
   - 'DOMAIN-SUFFIX,kat.cr,MyProxy'
   - 'DOMAIN-SUFFIX,klip.me,MyProxy'
   - 'DOMAIN-SUFFIX,libsyn.com,MyProxy'
   - 'DOMAIN-SUFFIX,linkedin.com,MyProxy'
   - 'DOMAIN-SUFFIX,line-apps.com,MyProxy'
   - 'DOMAIN-SUFFIX,linode.com,MyProxy'
   - 'DOMAIN-SUFFIX,lithium.com,MyProxy'
   - 'DOMAIN-SUFFIX,littlehj.com,MyProxy'
   - 'DOMAIN-SUFFIX,live.com,MyProxy'
   - 'DOMAIN-SUFFIX,live.net,MyProxy'
   - 'DOMAIN-SUFFIX,livefilestore.com,MyProxy'
   - 'DOMAIN-SUFFIX,llnwd.net,MyProxy'
   - 'DOMAIN-SUFFIX,macid.co,MyProxy'
   - 'DOMAIN-SUFFIX,macromedia.com,MyProxy'
   - 'DOMAIN-SUFFIX,macrumors.com,MyProxy'
   - 'DOMAIN-SUFFIX,mashable.com,MyProxy'
   - 'DOMAIN-SUFFIX,mathjax.org,MyProxy'
   - 'DOMAIN-SUFFIX,medium.com,MyProxy'
   - 'DOMAIN-SUFFIX,mega.co.nz,MyProxy'
   - 'DOMAIN-SUFFIX,mega.nz,MyProxy'
   - 'DOMAIN-SUFFIX,megaupload.com,MyProxy'
   - 'DOMAIN-SUFFIX,microsofttranslator.com,MyProxy'
   - 'DOMAIN-SUFFIX,mindnode.com,MyProxy'
   - 'DOMAIN-SUFFIX,mobile01.com,MyProxy'
   - 'DOMAIN-SUFFIX,modmyi.com,MyProxy'
   - 'DOMAIN-SUFFIX,msedge.net,MyProxy'
   - 'DOMAIN-SUFFIX,myfontastic.com,MyProxy'
   - 'DOMAIN-SUFFIX,name.com,MyProxy'
   - 'DOMAIN-SUFFIX,nextmedia.com,MyProxy'
   - 'DOMAIN-SUFFIX,nsstatic.net,MyProxy'
   - 'DOMAIN-SUFFIX,nssurge.com,MyProxy'
   - 'DOMAIN-SUFFIX,nyt.com,MyProxy'
   - 'DOMAIN-SUFFIX,nytimes.com,MyProxy'
   - 'DOMAIN-SUFFIX,omnigroup.com,MyProxy'
   - 'DOMAIN-SUFFIX,onedrive.com,MyProxy'
   - 'DOMAIN-SUFFIX,onenote.com,MyProxy'
   - 'DOMAIN-SUFFIX,ooyala.com,MyProxy'
   - 'DOMAIN-SUFFIX,openvpn.net,MyProxy'
   - 'DOMAIN-SUFFIX,openwrt.org,MyProxy'
   - 'DOMAIN-SUFFIX,orkut.com,MyProxy'
   - 'DOMAIN-SUFFIX,osxdaily.com,MyProxy'
   - 'DOMAIN-SUFFIX,outlook.com,MyProxy'
   - 'DOMAIN-SUFFIX,ow.ly,MyProxy'
   - 'DOMAIN-SUFFIX,paddleapi.com,MyProxy'
   - 'DOMAIN-SUFFIX,parallels.com,MyProxy'
   - 'DOMAIN-SUFFIX,parse.com,MyProxy'
   - 'DOMAIN-SUFFIX,pdfexpert.com,MyProxy'
   - 'DOMAIN-SUFFIX,periscope.tv,MyProxy'
   - 'DOMAIN-SUFFIX,pinboard.in,MyProxy'
   - 'DOMAIN-SUFFIX,pinterest.com,MyProxy'
   - 'DOMAIN-SUFFIX,pixelmator.com,MyProxy'
   - 'DOMAIN-SUFFIX,pixiv.net,MyProxy'
   - 'DOMAIN-SUFFIX,playpcesor.com,MyProxy'
   - 'DOMAIN-SUFFIX,playstation.com,MyProxy'
   - 'DOMAIN-SUFFIX,playstation.com.hk,MyProxy'
   - 'DOMAIN-SUFFIX,playstation.net,MyProxy'
   - 'DOMAIN-SUFFIX,playstationnetwork.com,MyProxy'
   - 'DOMAIN-SUFFIX,pushwoosh.com,MyProxy'
   - 'DOMAIN-SUFFIX,rime.im,MyProxy'
   - 'DOMAIN-SUFFIX,servebom.com,MyProxy'
   - 'DOMAIN-SUFFIX,sfx.ms,MyProxy'
   - 'DOMAIN-SUFFIX,shadowsocks.org,MyProxy'
   - 'DOMAIN-SUFFIX,sharethis.com,MyProxy'
   - 'DOMAIN-SUFFIX,shazam.com,MyProxy'
   - 'DOMAIN-SUFFIX,skype.com,MyProxy'
   - 'DOMAIN-SUFFIX,smartdnsMyProxy.com,MyProxy'
   - 'DOMAIN-SUFFIX,smartmailcloud.com,MyProxy'
   - 'DOMAIN-SUFFIX,sndcdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,sony.com,MyProxy'
   - 'DOMAIN-SUFFIX,soundcloud.com,MyProxy'
   - 'DOMAIN-SUFFIX,sourceforge.net,MyProxy'
   - 'DOMAIN-SUFFIX,spotify.com,MyProxy'
   - 'DOMAIN-SUFFIX,squarespace.com,MyProxy'
   - 'DOMAIN-SUFFIX,sstatic.net,MyProxy'
   - 'DOMAIN-SUFFIX,st.luluku.pw,MyProxy'
   - 'DOMAIN-SUFFIX,stackoverflow.com,MyProxy'
   - 'DOMAIN-SUFFIX,startpage.com,MyProxy'
   - 'DOMAIN-SUFFIX,staticflickr.com,MyProxy'
   - 'DOMAIN-SUFFIX,steamcommunity.com,MyProxy'
   - 'DOMAIN-SUFFIX,symauth.com,MyProxy'
   - 'DOMAIN-SUFFIX,symcb.com,MyProxy'
   - 'DOMAIN-SUFFIX,symcd.com,MyProxy'
   - 'DOMAIN-SUFFIX,tapbots.com,MyProxy'
   - 'DOMAIN-SUFFIX,tapbots.net,MyProxy'
   - 'DOMAIN-SUFFIX,tdesktop.com,MyProxy'
   - 'DOMAIN-SUFFIX,techcrunch.com,MyProxy'
   - 'DOMAIN-SUFFIX,techsmith.com,MyProxy'
   - 'DOMAIN-SUFFIX,thepiratebay.org,MyProxy'
   - 'DOMAIN-SUFFIX,theverge.com,MyProxy'
   - 'DOMAIN-SUFFIX,time.com,MyProxy'
   - 'DOMAIN-SUFFIX,timeinc.net,MyProxy'
   - 'DOMAIN-SUFFIX,tiny.cc,MyProxy'
   - 'DOMAIN-SUFFIX,tinypic.com,MyProxy'
   - 'DOMAIN-SUFFIX,tmblr.co,MyProxy'
   - 'DOMAIN-SUFFIX,todoist.com,MyProxy'
   - 'DOMAIN-SUFFIX,trello.com,MyProxy'
   - 'DOMAIN-SUFFIX,trustasiassl.com,MyProxy'
   - 'DOMAIN-SUFFIX,tumblr.co,MyProxy'
   - 'DOMAIN-SUFFIX,tumblr.com,MyProxy'
   - 'DOMAIN-SUFFIX,tweetdeck.com,MyProxy'
   - 'DOMAIN-SUFFIX,tweetmarker.net,MyProxy'
   - 'DOMAIN-SUFFIX,twitch.tv,MyProxy'
   - 'DOMAIN-SUFFIX,txmblr.com,MyProxy'
   - 'DOMAIN-SUFFIX,typekit.net,MyProxy'
   - 'DOMAIN-SUFFIX,ubertags.com,MyProxy'
   - 'DOMAIN-SUFFIX,ublock.org,MyProxy'
   - 'DOMAIN-SUFFIX,ubnt.com,MyProxy'
   - 'DOMAIN-SUFFIX,ulyssesapp.com,MyProxy'
   - 'DOMAIN-SUFFIX,urchin.com,MyProxy'
   - 'DOMAIN-SUFFIX,usertrust.com,MyProxy'
   - 'DOMAIN-SUFFIX,v.gd,MyProxy'
   - 'DOMAIN-SUFFIX,v2ex.com,MyProxy'
   - 'DOMAIN-SUFFIX,vimeo.com,MyProxy'
   - 'DOMAIN-SUFFIX,vimeocdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,vine.co,MyProxy'
   - 'DOMAIN-SUFFIX,vivaldi.com,MyProxy'
   - 'DOMAIN-SUFFIX,vox-cdn.com,MyProxy'
   - 'DOMAIN-SUFFIX,vsco.co,MyProxy'
   - 'DOMAIN-SUFFIX,vultr.com,MyProxy'
   - 'DOMAIN-SUFFIX,w.org,MyProxy'
   - 'DOMAIN-SUFFIX,w3schools.com,MyProxy'
   - 'DOMAIN-SUFFIX,webtype.com,MyProxy'
   - 'DOMAIN-SUFFIX,wikiwand.com,MyProxy'
   - 'DOMAIN-SUFFIX,wikileaks.org,MyProxy'
   - 'DOMAIN-SUFFIX,wikimedia.org,MyProxy'
   - 'DOMAIN-SUFFIX,wikipedia.com,MyProxy'
   - 'DOMAIN-SUFFIX,wikipedia.org,MyProxy'
   - 'DOMAIN-SUFFIX,windows.com,MyProxy'
   - 'DOMAIN-SUFFIX,windows.net,MyProxy'
   - 'DOMAIN-SUFFIX,wire.com,MyProxy'
   - 'DOMAIN-SUFFIX,wordpress.com,MyProxy'
   - 'DOMAIN-SUFFIX,workflowy.com,MyProxy'
   - 'DOMAIN-SUFFIX,wp.com,MyProxy'
   - 'DOMAIN-SUFFIX,wsj.com,MyProxy'
   - 'DOMAIN-SUFFIX,wsj.net,MyProxy'
   - 'DOMAIN-SUFFIX,xda-developers.com,MyProxy'
   - 'DOMAIN-SUFFIX,xeeno.com,MyProxy'
   - 'DOMAIN-SUFFIX,xiti.com,MyProxy'
   - 'DOMAIN-SUFFIX,yahoo.com,MyProxy'
   - 'DOMAIN-SUFFIX,yimg.com,MyProxy'
   - 'DOMAIN-SUFFIX,ying.com,MyProxy'
   - 'DOMAIN-SUFFIX,yoyo.org,MyProxy'
   - 'DOMAIN-SUFFIX,ytimg.com,MyProxy'
   - 'DOMAIN-SUFFIX,telegra.ph,MyProxy'
   - 'DOMAIN-SUFFIX,telegram.org,MyProxy'
   - 'IP-CIDR,91.108.4.0/22,MyProxy,no-resolve'
   - 'IP-CIDR,91.108.8.0/21,MyProxy,no-resolve'
   - 'IP-CIDR,91.108.16.0/22,MyProxy,no-resolve'
   - 'IP-CIDR,91.108.56.0/22,MyProxy,no-resolve'
   - 'IP-CIDR,149.154.160.0/20,MyProxy,no-resolve'
   - 'IP-CIDR6,2001:67c:4e8::/48,MyProxy,no-resolve'
   - 'IP-CIDR6,2001:b28:f23d::/48,MyProxy,no-resolve'
   - 'IP-CIDR6,2001:b28:f23f::/48,MyProxy,no-resolve'
   - 'IP-CIDR,120.232.181.162/32,MyProxy,no-resolve'
   - 'IP-CIDR,120.241.147.226/32,MyProxy,no-resolve'
   - 'IP-CIDR,120.253.253.226/32,MyProxy,no-resolve'
   - 'IP-CIDR,120.253.255.162/32,MyProxy,no-resolve'
   - 'IP-CIDR,120.253.255.34/32,MyProxy,no-resolve'
   - 'IP-CIDR,120.253.255.98/32,MyProxy,no-resolve'
   - 'IP-CIDR,180.163.150.162/32,MyProxy,no-resolve'
   - 'IP-CIDR,180.163.150.34/32,MyProxy,no-resolve'
   - 'IP-CIDR,180.163.151.162/32,MyProxy,no-resolve'
   - 'IP-CIDR,180.163.151.34/32,MyProxy,no-resolve'
   - 'IP-CIDR,203.208.39.0/24,MyProxy,no-resolve'
   - 'IP-CIDR,203.208.40.0/24,MyProxy,no-resolve'
   - 'IP-CIDR,203.208.41.0/24,MyProxy,no-resolve'
   - 'IP-CIDR,203.208.43.0/24,MyProxy,no-resolve'
   - 'IP-CIDR,203.208.50.0/24,MyProxy,no-resolve'
   - 'IP-CIDR,220.181.174.162/32,MyProxy,no-resolve'
   - 'IP-CIDR,220.181.174.226/32,MyProxy,no-resolve'
   - 'IP-CIDR,220.181.174.34/32,MyProxy,no-resolve'
   - 'DOMAIN,injections.adguard.org,DIRECT'
   - 'DOMAIN,local.adguard.org,DIRECT'
   - 'DOMAIN-SUFFIX,local,DIRECT'
   - 'IP-CIDR,127.0.0.0/8,DIRECT'
   - 'IP-CIDR,172.16.0.0/12,DIRECT'
   - 'IP-CIDR,192.168.0.0/16,DIRECT'
   - 'IP-CIDR,10.0.0.0/8,DIRECT'
   - 'IP-CIDR,17.0.0.0/8,DIRECT'
   - 'IP-CIDR,100.64.0.0/10,DIRECT'
   - 'IP-CIDR,224.0.0.0/4,DIRECT'
   - 'IP-CIDR6,fe80::/10,DIRECT'
   - 'DOMAIN-SUFFIX,cn,DIRECT'
   - 'DOMAIN-KEYWORD,-cn,DIRECT'
   - 'GEOIP,CN,DIRECT'
   - 'MATCH,MyProxy'
```

为了方便分享，可以把上面的yaml放在http服务器上，方便根据链接下载到任意设备上，还可以通过链接，定期刷新这个yaml文件。

## 第七步：设置系统代理

设置系统代理，以便使用 Clash 提供的代理端口，通常是 HTTP 代理端口 7890 和 SOCKS5 代理端口 7891。

## 第八步：测试代理连接

测试代理是否工作正常，例如访问 `http://ip-api.com/json` 检查出口 IP 是否已更改。

## 排查错误信息的方法

1. **查看 V2Ray 日志**：

```shell
sudo cat /var/log/v2ray/error.log
sudo cat /var/log/v2ray/access.log
```
2. **检查配置文件格式**：

```shell
sudo jq . /usr/local/etc/v2ray/config.json
```
3. **确认端口未被占用**：

```shell
sudo netstat -tuln | grep <PORT>
```
4. **检查防火墙设置**：

```shell
sudo firewall-cmd --zone=public --add-port=12345/tcp --permanent
sudo firewall-cmd --reload
```
5. **尝试手动启动 V2Ray**：

```shell
sudo /usr/bin/v2ray/v2ray -config /usr/local/etc/v2ray/config.json
```
6. **查看系统日志**：

```shell
sudo journalctl -u v2ray
```
## 安装 JQ

在基于 RHEL 的系统：

```shell
sudo yum install jq
```

使用 `jq` 检查 JSON 文件格式：

```shell
jq . /path/to/your/config.json
```

根据上述步骤操作后，应该能够成功安装并配置 Clash 使用 V2Ray 服务器。如果遇到问题，请根据排查错误信息的方法进行检查。



## 相关链接

https://clashverge.net/

