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
cp rootCA.pem assets/client/tls/ngrokroot.crt
cp device.crt assets/server/tls/snakeoil.crt
cp device.key assets/server/tls/snakeoil.key
~~~

### 4. 编译不同平台的服务端和客户端

~~~
# 编译64位linux平台服务端
GOOS=linux GOARCH=amd64 make release-server
# 编译64位windows客户端
GOOS=windows GOARCH=amd64 make release-server
# 如果是mac系统，GOOS=darwin。如果是32位，GOARCH=386
~~~

执行后会在ngrok/bin目录及其子目录下看到服务端ngrokd和客户端ngrok.exe。  


### 启动服务端

> http  
~~~
# 指定我们刚才设置的域名，指定http, https, tcp端口号，端口号不要跟其他程序冲突
./bin/ngrokd -domain="$NGROK_DOMAIN" -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":443"
~~~

> https  
~~~
#bin/ngrokd -domain="www.aiesst.com" -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":443" -tlsKey=./device.key -tlsCrt=./device.crt
~~~

> httpAddr,httpsAddr可进行自定义以避免与其他程序产生冲突  
> tunnelAddr: 默认值为443,可进行自定义以避免与其他程序产生冲突，客户端server_addr的端口号需与此一致  

### ngrok 加入系统服务 开机启动

#### 新建文件 /etc/systemd/system/ngrok.service  

~~~
[Unit]
Description=ngrok
After=network.target

[Service]
ExecStart=/usr/local/ngrok/bin/ngrokd -domain=$NGROK_DOMAIN -httpAddr=:80 -httpsAddr=:443 -tunnelAddr=:443 %i
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
server_addr: "YourDomain:443"
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

> server_addr: 服务端地址及端口，可根据需要进行自定义，端口需与服务端tunnelAddr一致，默认值为443  
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

### 常见问题

> 编译时在下面步骤卡住  

~~~
go get gopkg.in/yaml.v1
~~~

这是因为Git版本太低，请将服务器git版本升级到1.7.9.5以上。  

> ngrok首次编译时需要在国外网站下载一些依赖。可能会很慢甚至timeout。多尝试几次，或者以其他科学方式连国外网站。  
