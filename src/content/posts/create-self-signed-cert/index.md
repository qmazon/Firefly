---
title: 制作自定义CA及自签名SSL证书
pinned: false
description: “需要提供私钥才能安装证书”？那是你没有正确地创建CA证书！
tags: [网络, SSL]
category: 技术
author: 雪纷飞
draft: false
# updated: 2025-12-22
published: 2025-12-22
image: "api"
---

本文使用Ubuntu环境下的PowerShell 7来进行。Bash环境如法炮制即可。

基本流程：`生成 Root CA -> 生成 Server CSR -> 用 Root CA 签名 Server CSR`

# 约定

先执行下面的代码吧！可以自行设定相关文件的名称。

```ps1
# 基础目录名称
$RootDir = "ssl-work"
$CaDir = "$RootDir\ca"
$ServerDir = "$RootDir\server"

# CA文件名
$CaKeyName = "myRootCA.key"
$CaCertName = "myRootCA.crt"

# 服务器证书文件名
$ServerKeyName = "server.key"
$ServerCsrName = "server.csr"
$ServerCertName = "server.crt"
$ServerExtName = "cert.ext"

$CaKeyPath = "$CaDir\$CaKeyName"
$CaCertPath = "$CaDir\$CaCertName"

$ServerKeyPath = "$ServerDir\$ServerKeyName"
$ServerCsrPath = "$ServerDir\$ServerCsrName"
$ServerCertPath = "$ServerDir\$ServerCertName"
$ServerExtPath = "$ServerDir\$ServerExtName"

New-Item -ItemType Directory -Path $CaDir
New-Item -ItemType Directory -Path $ServerDir
```

**`crt`结尾的公钥，可以自由分发。`key`结尾的私钥，务必严格保护！**

# 建立自定义CA（Certificate Authority）

首先，我们需要成为一个权威机构（CA）。这一步只需要做一次。以后签发新证书时，直接使用这里生成的CA文件即可。同样地，远端设备只需要信任我们的CA证书，那么之后该CA签发的所有证书都会被信任。

先创建一个CA私钥。

```ps1
openssl genrsa -des3 -out $CaKeyPath 2048
```

执行命令后，会有如下提示，最好设置一个密码哦：

```txt
Enter pass phrase for myRootCA.key:
Verifying - Enter pass phrase for myRootCA.key:
```

:::caution
再次重申一遍！`key`十分重要！确保它的权限是`400`，所有者与所有组都是`root`！你只能分发以`crt`为拓展名的文件！
:::

接下来我们来生成公钥，有效期二十年。

```ps1
openssl req -x509 -new -nodes -key $CaKeyPath -sha256 -days 7300 -out $CaCertPath
```

填写一些基本数据（请自行替换被高亮标记的内容！后同）：
```txt collapse={1-7} ins="Beijing" ins="CN" ins="Snowy Personal" ins="Snowy Root CA" ins="snowflies@outlook.com"
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Beijing
Locality Name (eg, city) []:Beijing
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Snowy Personal
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:Snowy Root CA
Email Address []:snowflies@outlook.com
```

现在，请将`$CaCertPath`发送你的其他设备，并安装为“受信任的根证书颁发机构”！

# 为具体的域签发证书

接下来为你的具体的域（如`localhost`）生成证书。这里包含关键的Subject Alternative Name配置，确保Chrome、Firefox等浏览器不报错。

```ps1
# 生成私钥
openssl genrsa -out $ServerKeyPath 2048
# 生成证书签名请求CSR，
openssl req -new -key $ServerKeyPath -out $ServerCsrPath
```

在生成CSR的时候，会有如下流程，正常填写相关信息即可。

此外，这里一般不用创建密码，直接用空密码回车就行。

```txt collapse={1-7} ins="Beijing" ins="CN" ins="Snowy Personal" ins="Snowy Local Cert" ins="snowflies@outlook.com"
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Beijing
Locality Name (eg, city) []:Beijing
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Snowy Personal
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:Snowy Local Cert
Email Address []:snowflies@outlook.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

定义SAN。下面的这个会匹配`mysite.local`及其子域，以及一个纯IP。如果你认为你的服务不会/不允许被通过IP访问，把`IP.1`删去即可。

```ps1 {"请在这里填写你自己的域的相关信息：":7-10}
$ExtContent = @"
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]

DNS.1 = mysite.local
DNS.2 = *.mysite.local
IP.1 = 192.168.1.100
"@

Set-Content -Path $ServerExtPath -Value $ExtContent -Encoding Ascii
# 记得输入之前的密码
openssl x509 -req -in $ServerCsrPath -out $ServerCertPath -days 3650 `
  -CA $CaCertPath -CAkey $CaKeyPath `
  -CAcreateserial -extfile $ServerExtPath
```

好了，现在你可以正常使用这份证书了！nginx中直接把证书和私钥分别指向`$ServerCertPath`和`$ServerKeyPath`就行，只要客户端信任了你的`$CaCertPath`就不会报错。

# 安装证书

- 对于Windows设备，你需要双击`crt`文件，选择“安装证书”，安装到“本地计算机”，然后**务必选择“受信任的根证书颁发机构”**！
  :::note
  如果你让Windows来自动选择，那么它会帮你安装到“中间证书颁发机构”，安装了后证书依然不被信任。
  :::
- 对于Ubuntu，将`crt`文件复制到`/usr/local/share/ca-certificates/`并运行`sudo update-ca-certificates`即可。
- 对于安卓设备，**现代化的安卓系统不允许安装根证书**。
  :::tip
  如果你的设备安装了[Magisk][magisk]或者[Kernel SU][kernel-su]，那么你可以使用下面这个插件：
  ::github{repo="ys1231/MoveCertificate"}
  :::
- 对于MacOS，~**我不会，等我长大后再学习吧**~！
- 对于iOS，~**我不会，等我长大后再学习吧**~！

<!-- 引用 -->
[magisk]: https://github.com/topjohnwu/Magisk "Magisk"
[kernel-su]: https://kernelsu.org/ "Kernel SU"