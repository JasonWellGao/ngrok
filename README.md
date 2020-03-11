# 搭建ngrok内网穿透服务器

## 系统环境
> 公网linux服务器,本例采用Centos7  

## 域名
> 为便于访问,可将自有域名解析到服务器,并对子域名做泛解析  

## 服务端安装

### 1. 安装git,golang和openssl

~~~
yum install -y git golang openssl
~~~

如系统中已安装以上软件可不用重复安装,如原安装版本过低可查看可用版本并进行版本升级  

~~~
yum list YourAppName
yum update YourAppName
~~~

### 2.clone ngrok项目到服务器

> ngrok项目在github中有公开仓库,将其clone到服务器/usr/local/ngrok目录  

~~~
git clone https://github.com/inconshreveable/ngrok.git /usr/local/ngrok
~~~

### 3.生成证书

~~~
# 这里替换为自己的独立域名
export NGROK_DOMAIN="YourDomain"

#进入到ngrok目录生成证书
cd /usr/local/ngrok

# 下面的命令用于生成证书
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out device.key 2048
openssl req -new -key device.key -subj "/CN=$NGROK_DOMAIN" -out device.csr
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000

# 将我们生成的证书替换ngrok默认的证书
\cp rootCA.pem assets/client/tls/ngrokroot.crt -f
\cp device.crt assets/server/tls/snakeoil.crt -f
\cp device.key assets/server/tls/snakeoil.key -f
~~~

这里用的 ` \cp ` 命令，因为系统默认 cp 是有别名的，实际执行的是 cp -i，需要每一个覆盖的时候都确认一遍，加上 \cp 就可强制覆盖了。到这个地方，证书生成已经复制的准备工作就已经完成了。  

### 4. 编译不同平台的服务端和客户端

~~~
# 编译64位linux平台服务端
GOOS=linux GOARCH=amd64 make release-server
# 编译64位windows客户端
GOOS=windows GOARCH=amd64 make release-client
# 如果是mac系统，GOOS=darwin。如果是32位，GOARCH=386
~~~

执行后会在ngrok/bin目录及其子目录下看到服务端ngrokd和客户端ngrok.exe。  

不同平台使用不同的 GOOS 和 GOARCH，前面的编译选项就是指 go os , go 编译出来的操作系统 (windows,linux,darwin) ；go arch, 对应的构架 (386,amd64,arm)  
~~~
Linux 平台 32 位系统：GOOS=linux GOARCH=386
Linux 平台 64 位系统：GOOS=linux GOARCH=amd64
Windows 平台 32 位系统：GOOS=windows GOARCH=386
Windows 平台 64 位系统：GOOS=windows GOARCH=amd64
MAC 平台 32 位系统：GOOS=darwin GOARCH=386
MAC 平台 64 位系统：GOOS=darwin GOARCH=amd64
ARM 平台：GOOS=linux GOARCH=arm
~~~

通过前面的步骤，就会在bin目录里面生成所有的客户端文件，客户端平台是文件夹的名字，客户端放在对应的目录下，当前Linux平台客户端在bin目录下。然后我们就可以打个包，把所有文件下载到自己的本机了。  

### 启动服务端

~~~
cd /usr/local/ngrok
export NGROK_DOMAIN="YourDomain"
~~~

> http  
~~~
# 指定我们刚才设置的域名，指定http, https, tcp端口号，端口号不要跟其他程序冲突
./bin/ngrokd -domain="$NGROK_DOMAIN" -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":4443"
# 或
./bin/ngrokd -domain="YourDomain" -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":4443"
~~~

> https  
~~~
bin/ngrokd -domain="$NGROK_DOMAIN" -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":4443" -tlsKey=./device.key -tlsCrt=./device.crt
# 或
bin/ngrokd -domain="YourDomain" -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":4443" -tlsKey=./device.key -tlsCrt=./device.crt
~~~

> httpAddr,httpsAddr可进行自定义以避免与其他程序产生冲突  
> tunnelAddr: 默认值为4443,可进行自定义以避免与其他程序产生冲突，客户端server_addr的端口号需与此一致  

### ngrok 加入系统服务 开机启动

#### 新建文件 /etc/systemd/system/ngrok.service  

> http  

~~~
[Unit]
Description=ngrok
After=network.target

[Service]
ExecStart=/usr/local/ngrok/bin/ngrokd -domain=YourDomain -httpAddr=:80 -httpsAddr=:443 -tunnelAddr=:4443 %i
ExecStop=/usr/bin/killall ngrok

[Install]
WantedBy=multi-user.target
~~~

> https  

~~~
[Unit]
Description=ngrok
After=network.target

[Service]
ExecStart=/usr/local/ngrok/bin/ngrokd -domain=YourDomain -httpAddr=:80 -httpsAddr=:443 -tunnelAddr=:4443 -tlsKey=/usr/local/ngrok/device.key -tlsCrt=/usr/local/ngrok/device.crt %i
ExecStop=/usr/bin/killall ngrok

[Install]
WantedBy=multi-user.target
~~~

#### 重载系统服务

~~~
systemctl daemon-reload
~~~

#### 设置开机启动

~~~
systemctl enable ngrok.service
~~~

#### 启动服务

~~~
systemctl start ngrok.service
~~~

### 停止服务

~~~
systemctl stop ngrok.service
~~~

> 在修改ngrok.service后需重载服务并停止ngrok.service，再次启动ngrok.service才可生效  

### 客户端配置文件

#### 配置文件名
> ngrok.cfg  

#### 客户端示例配置

~~~
server_addr: "YourDomain:4443"
trust_host_root_certs: false
tunnels:
    http:
        proto:
            http: 80
        subdomain: YourSubDomain
    tomact:
        remote_port: 108080
        proto:
            tcp: 8080
    win:
        remote_port: 103389
        proto:
            tcp: 127.0.0.1:3389
~~~

> server_addr: 服务端地址及端口，可根据需要进行自定义，端口需与服务端tunnelAddr一致，默认值为4443  
> tunnels: 隧道设置,示例配置中对常用的80、8080以及3389端口进行配置  
> subdomain: 经测试同一客户端在配置一个subdmoain时可运行配置多个时无法运行，不同客户端subdomain不得相同否则会产生冲突导致客户端无法启动  


### 客户端启动文件

#### 启动文件名

> startup.bat 或其他自定义文件名bat文件名均可  

#### 启动文件示例


~~~
@echo on
cd %cd%
#ngrok -proto=tcp 22
#ngrok start web
ngrok -config=ngrok.cfg -log=ngrok.log start-all
~~~

> 本示例中设置ngrok配置从ngrok.cfg中读取,log写入到ngrok.log中,启动所有tunnels


### linux 客户端

#### 新建文件 /etc/systemd/system/ngrok.service  

~~~
[Unit]
Description=ngrok
After=network.target

[Service]
ExecStart=/YourClientDir/ngrok -config=/YourClientDir/ngrok.cfg -log=/YourClientDir/ngrok.log start-all %i
ExecStop=/usr/bin/killall ngrok

[Install]
WantedBy=multi-user.target
~~~

#### 设置启动方式

> 参照服务端设置启动方式进行设置  

### 常见问题

> 编译时在下面步骤卡住  

~~~
go get gopkg.in/yaml.v1
~~~

这是因为Git版本太低，请将服务器git版本升级到1.7.9.5以上。  

> ngrok首次编译时需要在国外网站下载一些依赖。可能会很慢甚至timeout。多尝试几次，或者以其他科学方式连国外网站。  

> 防火墙可能会影响ngrok的访问，如连接失败请检查防火墙状态及设置

~~~
# 查看firewall服务状态
systemctl status firewalld

# 查看firewall的状态
firewall-cmd --state

# 开启停止防火墙
# 开机启动：
systemctl enable firewalld.service

# 启动：
systemctl start firewalld.service

# 停止：
systemctl stop firewalld.service
# 禁止开机启动：
systemctl disable firewalld.service

# 开放端口
firewall-cmd --zone=public --add-port=80/tcp --permanent
# –zone #作用域
# –add-port=80/tcp #添加端口，格式为：端口号/协议
# –permanent #永久生效，没有此参数重启后失效

#禁用端口
firewall-cmd --zone=public --remove-port=80/tcp --permanent

# 应用修改(修改配置后要重启防火墙)
firewall-cmd --reload

# 查看所有开放的端口
firewall-cmd --zone=dmz --list-ports
firewall-cmd --list-ports
~~~
