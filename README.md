# 在海外区域使用AWS Client VPN访问没有Internet Gateway的VPC内网资源

## 一、背景和架构

AWS Client VPN是一种基于客户端的托管VPN服务，让您能够安全地访问云上资源或借助云作为网络通道访问其他资源。AWS Client VPN具有专用的VPN客户端，也支持使用OpenVPN作为客户端。

注意：AWS Client VPN只能用于企业海外员工到附近的海外AWS区域的访问接入，不可用于跨境访问。跨境需求需要申请符合相关法律规定的具有资质的跨境专线。

与云上自行部署Client VPN相比，AWS Client VPN除了具有运维管理的便捷性之外，还可以用于与没有Internet Gateway的VPC内网互联实现对云上私密数据的访问，且访问VPN同时不影响客户端本机的互联网访问。如下图架构所示。

![](https://myworkshop.bitipcman.com/client/client00.png)

在上图中，位于海外区域的远程用户希望访问云上的VPC内的应用，且出于合规安全考虑云上的VPC是一个没有关联Internet Gateway、也没有外网路由的内部VPC。此时AWS Client VPN支持通过ENI方式将流量注入到VPC内实现访问。同时，为了确保客户端能对其他Internet网络正常访问，本方案将使用AWS Client VPN的Split-tunnul功能分离网络流量，允许去往非VPC的流量直接从客户端对外发出而不经过VPNs。本文描述此场景下的配置过程。

## 二、创建AWS Client VPN需要的证书

AWS Client VPN使用的是OpenVPN协议，身份认证方式支持微软AD认证和SSL证书认证等方式。本文采用证书方式认证进行配置。

### 1、创建CA和服务器证书

首先在海外AWS区域创建一个EC2，建议使用Amazon Linux 2系统，在其上配置AWSCLI工具和对应的AKSK，使其具有AWS Certificate Manager(ACM)服务操作权限。

执行如下命令生成CA证书。注意最后一个命令执行后，需要输入证书使用的域名。

```
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
./easyrsa init-pki
./easyrsa build-ca nopass
```

接下来生成服务器证书和密钥，执行如下命令。

```
./easyrsa build-server-full server nopass
```

由此将获得文件名为`server.crt`证书和`server.key`密钥。

### 2、创建客户端证书

生成客户端证书和密钥，替换其中的前缀username为实际使用的用户名，替换域名domain.tld为实际域名。执行如下命令。

```
./easyrsa build-client-full username.domain.tld nopass
```

需要注意，未来新建用户时候都需要再次执行本命令生成新的证书。如果觉得生成为每个用户单独生成证书不方便操作，可参考使用微软AD认证进行用户名和密码认证。

### 3、导出证书并上传到ACM上

执行如下命令。请注意替换实际的文件名，并且服务器端证书和客户端证书都需要上传。

```
mkdir ~/custom_folder/
cp pki/ca.crt ~/custom_folder/
cp pki/issued/server.crt ~/custom_folder/
cp pki/private/server.key ~/custom_folder/
cp pki/issued/username.domain.tld.crt ~/custom_folder
cp pki/private/username.domain.tld.key ~/custom_folder/
cd ~/custom_folder/
```

接下来通过AWSCLI上传证书到ACM。在执行这一步之前，请确认CLI已经配置好对应的region，且具有操作ACM证书服务的权限。注意ACM是以区域级别服务，请确认将证书上传到要使用AWS Client VPN的region。本文以`eu-central-1`为例。执行如下命令：

```
aws acm import-certificate --certificate fileb://server.crt --private-key fileb://server.key --certificate-chain fileb://ca.crt --region eu-central-1
aws acm import-certificate --certificate fileb://username.domain.tld.crt --private-key fileb://username.domain.tld.key --certificate-chain fileb://ca.crt --region eu-central-1
```

返回CertificateArn信息则表示成功。

## 三、创建AWS Client VPN终端节点

### 1、创建Endpoint终端节点

进入对应区域的VPC服务界面，在左侧菜单中找到`Client VPN Endpoints`，点击进入，并点击新建，如下截图。

![](https://myworkshop.bitipcman.com/client/client01.png)

在创建界面上填写VPN服务名称、描述信息，并设置IP地址段。IP地址段是AWS Client VPN服务器与客户端之间的一个私有网络，这个地址段的子网掩码范围/12和/22之间，且地址不能与客户端本地或云上VPC冲突。例如本文使用`10.88.0.0/16`作为地址段。

然后在下方的服务器证书位置，选择ACM中包含的server证书。如下截图。

![](https://myworkshop.bitipcman.com/client/client09.png)

在客户端认证选项中，选择`Use mutual authentication`使用证书双向认证，然后在Client证书选择框中，选择ACM中包含的client证书。如下截图。

![](https://myworkshop.bitipcman.com/client/client10.png)

接下来将页面向下移动。在DNS位置留空，VPN服务器将不强制推送DNS服务器。在对接企业AD场景下如果有需要的话可推送自己的DNS。在传输协议位置保持默认的UDP协议。在`Enable split-tunnel`位置，选中这个选项，表示客户端将把去往AWS的流量和本地互联网流量分离。由此不需要所有流量都经过VPN处理。接下来选择安全组为本VPC默认的安全组，默认的安全组规则是对外出栈放行，对内入栈禁止所有。如下截图。

![](https://myworkshop.bitipcman.com/client/client11.png)

页面继续向下移动，在页面下方，选中`Do you want to enable Client Login Banner text`的选项，并设置一段文本，可以在VPN连接过程中显示这个文本信息。最后点击右下角创建按钮。如下截图。

![](https://myworkshop.bitipcman.com/client/client12.png)

至此创建Client VPN Endpoint完成。

### 2、绑定子网

进入创建好的AWS Client VPN Endpoint界面，找到第二个标签页`Associations`，点击`Associate`按钮。如下截图。

![](https://myworkshop.bitipcman.com/client/client13.png)

在弹出的绑定子网界面中，选中AWS Client VPN要连接的子网。首先选择VPC和第一个子网。然后点确定。如下截图。

![](https://myworkshop.bitipcman.com/client/client14.png)

重复以上步骤，将两个子网都绑定到AWS Client VPN。如下截图。

![](https://myworkshop.bitipcman.com/client/client15.png)

接下来需要等待几分钟，等待黄色的`Associating`字样变成绿色的`Associated`。如下截图。

![](https://myworkshop.bitipcman.com/client/client16.png)

### 3、绑定策略

进入创建好的AWS Client VPN Endpoint界面，找到第四个标签页`Authorization`授权界面。点击`Authorize Ingress`按钮。如下截图。

![](https://myworkshop.bitipcman.com/client/client17.png)

输入AWS Client VPN要访问的VPC的CIDR，例如本文的VPC地址段是`10.1.0.0/16`，然后点击授权。如下截图。

![](https://myworkshop.bitipcman.com/client/client18.png)

至此VPC服务器端配置完成。

## 四、获取客户端配置文件并修改配置

### 1、下载客户端配置文件

在上一步完成操作的界面，点击页面中间蓝色的按钮`Download Client Configuration`，获取客户端配置文件。

![](https://myworkshop.bitipcman.com/client/client19.png)

这样将在下载目录中打开名为`downloaded-client-config.ovpn`的文件，接下来使用任意文本编辑器修改其配置。

### 2、设置客户端证书

接上一步，打开扩展名为ovpn的配置文件，在最后增加如下两段内容：

```
<cert>
这里粘贴上前文生成的客户端密钥username.domain.tld.crt的内容
</cert>

<key>
这里粘贴上前文生成的客户端密钥username.domain.tld.key的内容
</key>
```

### 3、设置访问域名

接上一步，找到配置文件中开头部分如下一行。

```
remote cvpn-endpoint-05ba461a63c340555.prod.clientvpn.eu-central-1.amazonaws.com 443
```

在其前方增加一个子域名，使用前边申请证书的子域名即可，例如`username.domain.tld`。如下配置。

```
remote username.cvpn-endpoint-05ba461a63c340555.prod.clientvpn.eu-central-1.amazonaws.com 443
```

保存退出，至此本用户的客户端配置文件修改完成。现在需要将此文件复制到要连接VPN的客户机上。

请注意，今后每增加一个新的用户，都需要新制作这样一个客户端配置文件。由此，当单个用户需要注销的时候，只要在ACM服务内注销掉用户对应的客户端证书即可。

## 五、下载客户端并连接VPN

在要使用VPN的客户机上下载AWS Client VPN客户端对应的操作系统的版本，包括Windows、Linux和MacOS的支持。如下截图。

[https://aws.amazon.com/vpn/client-vpn-download/](https://aws.amazon.com/vpn/client-vpn-download/)

![](https://myworkshop.bitipcman.com/client/client07.png)

安装完成后启动客户端。点击菜单上的`File`命令，然后点击`Manage Profiles`按钮管理配置文件。如下截图。

![](https://myworkshop.bitipcman.com/client/client02.png)

点击添加按钮，再选择前文编辑完成的客户端配置文件，然后点击`Add Profile`按钮完成添加。如下截图。

![](https://myworkshop.bitipcman.com/client/client03.png)

添加配置文件完成。此时可以准备开始连接了。如下截图。

![](https://myworkshop.bitipcman.com/client/client04.png)

连接成功后，使用cmd去ping在VPC内网的环境，可看到ping成功。如下截图。

![](https://myworkshop.bitipcman.com/client/client06.png)

此外，通过AWS控制台上Client VPN界面的`Connections`标签页，也可以看到当前连接的客户端。如下截图。

![](https://myworkshop.bitipcman.com/client/client08.png)

在前文配置VPN过程中已经将`Split-tunnel`选项设置为`Enable`启用状态，所以只有去往VPC中的流量才会通过VPN传输，其他流量将从客户本地网络上直接访问互联网。由于本次配置的云上VPC是没有Internet Gateway的，因此当您连接到Client VPN后能同时访问VPC和访问Internet，就意味着启用了`Split-tunnel`分流功能工作正常。

至此配置完成。

## 六、总结

从以上配置可以看出，AWS Client VPN可支持海外用户就近接入AWS区域的VPC网络访问内网应用系统，还可通过策略设置，管控访问范围。

本文使用了证书认证方式，每次新增用户时候都需要为新用户生成新的客户端证书，将证书上传到ACM并修改用户端配置文件。此过程，可通过AWSCLI、Shell脚本等一系列服务与企业现有IT系统对接和集成完成自动Provisioning。此过程不在本文讨论之列。

注意：AWS Client VPN只能用于企业海外员工到附近的海外AWS区域的访问接入，不可用于跨境访问。跨境需求需要申请符合相关法律规定的具有资质的跨境专线。

全文完。