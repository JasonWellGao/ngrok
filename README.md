


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

