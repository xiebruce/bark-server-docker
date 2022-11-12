# 使用说明

## 环境准备
### 工具材料
- 1、准备一台Linux服务器（最好是国外服务器，这样不用备案更省事，这里以Debian 11系统为例），假设ip为：12.34.56.78；
- 2、准备一个域名，假设为：zhangsan.com；
- 3、解析一个三级域名到到你的服务器，假设为：bark.zhangsan.com，添加一条A记录，解析到你的服务器：12.34.56.78，并且`ping bark.zhangsan.com`能ping通（当然你也可以解析多个域名到同一个服务器，比如如果是使用cloudflare解析，我一般会再解析一个`bark-cdn.zhangsan.com`到我的服务器中，这个要开启小云朵表示走cdn）；

## 服务器端操作
### 安装docker
在服务器中安装docker：请参考官方文档[Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/)。

### 下载bark-server
我准备了一个bark-server的docker运行环境，直接clone下来就行：
```bash
cd /root/
git clone https://github.com/xiebruce/bark-server-docker.git
```

## 申请https证书
申请https证书有以下两种方式：

- 1、dns-01方式：不能是.cf .ml .tk .ga .gq这些免费域名，并且需要添加环境变量(前面有说过)；
- 2、http-01方式：任何域名都可以，不用添加环境变量，但不能申请通配符证书(不推荐，除非你是免费域名)；

**建议**：如果你的域名不是.cf .ml .tk .ga .gq这些免费域名，建议使用第一种方式。

### dns-01方式申请证书
执行以下命令添加环境变量(如果你用的是zsh，需要把.bashrc修改为.zshrc)，这里假设你是在cloudflare解析的域名
```bash
cat > ~/.bashrc << EOF
export CF_Email=zhangsan@163.com
export CF_Key=sdf39sdDfsjDkJN#)EJD
EOF
```

注意：执行上边命令前，要把CF_Email和CF_Key修改为你自己的，而且key未必是CF_Email和CF_Key，具体要看你是在哪个网站中解析你的域名的，比如如果是在阿里云解析域名，则需要变成
```bash
cat > ~/.bashrc << EOF
export Ali_Key=zhangsan@163.com
export Ali_Secret=sdf39sdDfsjDkJN#)EJD
EOF
```

其它的请自己从[这里](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi)找到 对应文件，然后点进去看源码里用的哪个变量。

如果添加错了，请自己用vi/vim/nano命令去~/.bashrc文件或~/.zshrc文件中修改

---

让环境变量生效(如果你用的是zsh，要把下边的.bashrc换成.zshrc)
```bash
source ~/.bashrc
```

确认环境变量确实已生效，如果生效，运行以下两个会输出你前面填到
```bash
echo $CF_Email
echo $CF_Key
```

运行acme.sh容器
```bash
# 进入docker-compose.yml所在目录
cd bark-server-docker

# 运行acme.sh容器
docker compose up -d acme.sh

# 查看容器是否成功启动
docker ps
```

如果容器成功启动，显示应该类似如下
```bash
CONTAINER ID   IMAGE           COMMAND                  CREATED         STATUS         PORTS     NAMES
2038321f125f   bruce/acme.sh   "/usr/sbin/crond -f …"   7 seconds ago   Up 6 seconds             acme.sh
```

acme.sh容器启动后，进入acme.sh容器中
```bash
docker exec -it acme.sh sh
```

在容器中执行以下命令获取证书(别急着复制粘贴，要改参数，往下看)
```bash
acme.sh --issue --dns dns_cf -d zhangsan.com -d '*.zhangsan.com' --keylength ec-256
```
- `--dns dns_cf`表示你的域名是在cloudflare解析的，如果你是阿里云解析的，那就是`--dns dns_ali`，如果是其它服务商解析的，自己从[这里](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi)找，注意这个要与前面的环境变量对应；
- 命令中两处`zhangsan.com`要换成你自己的域名；

申请成功后，安装证书
```bash
acme.sh --install-cert --ecc --home /root/acmeout -d 'zhangsan.com' \
--key-file       /root/certs/private.pem  \
--fullchain-file /root/certs/fullchain.pem
```

安装成功是这样的
```bash
[Fri Nov 11 21:57:17 CST 2022] Installing key to: /root/certs/private.pem
[Fri Nov 11 21:57:17 CST 2022] Installing full chain to: /root/certs/fullchain.pem
```

退出到宿主机(因为刚刚是在容器中)
```bash
exit
```

### http-01方式申请证书
**注意**：该方式与前面的dns-01方式不要同时使用，根据你自己的情况，你选其中一种方式来申请即可。

---

进入以下目录
```bash
cd bark-server-docker/conf/nginx/vhost
```

可以看到目录中有这两个文件
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
```

复制一份bark.example.com-80.conf.bak，并修改文件名如下(其中zhangsan.com要改成你自己真实的域名)
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
bark.zhangsan.com-80.conf
```
注意：暂时不要复制`bark.example.com.conf.bak`，因为还没有申请证书，你启用了它就会报错。

打开`bark.zhangsan.com-80.conf`，把里面的`example.com`修改成`zhangsan.com`(当然你要用你自己的域名)，其实主要就是`server_name`,`access_log`和`error_log`这三个
```bash
server_name bark.zhangsan.com bark-cdn.zhangsan.com;

# access_log /data/wwwlogs/bark.zhangsan.com_nginx.access.log combined;
access_log /data/wwwlogs/bark.zhangsan.com_nginx.access.log combined buffer=1k;
error_log /data/wwwlogs/bark.zhangsan.com_nginx.error.log error;
```

运行所有服务(需要确保80,8080这两个端口未被占用)
```bash
docker compose up -d
```

运行后，使用`docker ps`查看，应该是如下所示这样的
```bash
CONTAINER ID   IMAGE               COMMAND                  CREATED          STATUS          PORTS     NAMES
aa969e57ba73   bruce/nginx         "/docker-entrypoint.…"   37 seconds ago   Up 1 second               nginx
a912a1d54baa   finab/bark-server   "/entrypoint.sh bark…"   37 seconds ago   Up 37 seconds             bark-server
2038321f125f   bruce/acme.sh       "/usr/sbin/crond -f …"   45 minutes ago   Up 45 minutes             acme.sh
```

进入acme.sh容器中
```bash
docker exec -it acme.sh sh
```

运行以下命令申请证书(注意每个`-d`表示一个域名，如果你只有`bark.zhangsan.com`一个域名，那后面那个`-d bark-cdn.zhangsan.com`就可以删掉)，当然`zhangsan.com`要换成你自己的真实域名
```bash
acme.sh --issue -d bark.zhangsan.com -d bark-cdn.zhangsan.com --webroot /data/wwwroot/well-known/  --home /root/acmeout --keylength ec-256
```

申请证书成功后，安装证书
```bash
acme.sh --install-cert --ecc --home /root/acmeout -d 'bark.zhangsan.com' \
--key-file       /root/certs/private.pem  \
--fullchain-file /root/certs/fullchain.pem
```

安装成功是这样的
```bash
[Fri Nov 11 21:57:17 CST 2022] Installing key to: /root/certs/private.pem
[Fri Nov 11 21:57:17 CST 2022] Installing full chain to: /root/certs/fullchain.pem
```

## 运行所有服务
### 修改nginx配置文件
进入以下目录
```bash
cd bark-server-docker/conf/nginx/vhost
```

可以看到目录中有这两个文件
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
```

把它们分别复制一份并修改文件名如下(其中zhangsan.com要改成你自己真实的域名)
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
bark.zhangsan.com-80.conf
bark.zhangsan.com.conf
```

打开前面两份配置文件，分别把这两个文件中的`example.com`改成`zhangsan.com`(当然你要用你真实的域名)，其实主要就是`server_name`,`access_log`和`error_log`这三个
```bash
server_name bark.zhangsan.com bark-cdn.zhangsan.com;

# access_log /data/wwwlogs/bark.zhangsan.com_nginx.access.log combined;
access_log /data/wwwlogs/bark.zhangsan.com_nginx.access.log combined buffer=1k;
error_log /data/wwwlogs/bark.zhangsan.com_nginx.error.log error;
```

检查80,443,8080三个端口未被占用，因为80和443 nginx要使用，8080是bark-server的端口(无法改，代码中写死的，main.go文件中能搜到)
```bash
netstat -tlunp | grep 80
netstat -tlunp | grep 443
```

如果上边说的三个端口都没被占用，那么就可以启动所有服务了
```bash
# 进入docker-compose.yml所在文件夹
cd bark-server-docker

# 启动所有服务
docker compose up -d
```

运行之后，可以看到三个docker容器都启动了，如下所示
```bash
CONTAINER ID   IMAGE               COMMAND                  CREATED          STATUS          PORTS     NAMES
aa969e57ba73   bruce/nginx         "/docker-entrypoint.…"   37 seconds ago   Up 1 second               nginx
a912a1d54baa   finab/bark-server   "/entrypoint.sh bark…"   37 seconds ago   Up 37 seconds             bark-server
2038321f125f   bruce/acme.sh       "/usr/sbin/crond -f …"   45 minutes ago   Up 45 minutes             acme.sh
```

### 测试
运行以下命令进行测试
```bash
curl https://bark.zhangsan.com/ping
```

如果一切正常，应该返回如下所示的json
```json
{"code":200,"message":"pong","timestamp":1668248534}
```

### 添加地址到Bark app
在Bark app中，点击底部第一个按钮“服务器” → 点击“右上角+号” → 把地址`https://bark.example.com`粘贴进去 → 点击右上角的对勾✓ → 它会返回到服务器列表中，至此就添加完成了。







[Fri Nov 11 20:12:04 CST 2022] Error, can not get domain token entry *.xiebruce.tk for http-01
[Fri Nov 11 20:12:04 CST 2022] The supported validation types are: dns-01 , but you specified: http-01
[Fri Nov 11 20:12:04 CST 2022] Please add '--debug' or '--log' to check more details.
[Fri Nov 11 20:12:04 CST 2022] See: https://github.com/acmesh-official/acme.sh/wiki/How-to-debug-acme.sh

### 确定
- 1、确定8080端口未被占用，因为bark-server是在代码里写死8080端口的，在[main.go](https://github.com/Finb/bark-server/blob/v2/main.go)中搜索`8080`就能搜索到；
- 2、进入bark-server/conf/nginx/vhost/文件夹，复制一份bark.example.com-80.conf.bak，并重命名为：bark.zhangsan.com-80.conf，并把该文件内部的所有域名都改为bark.zhangsan.com（注意bark.zhangsan.com是你自己的域名，如果你自己的域名不是这个，那你要改成你自己的）；
- 3、运行以下命令，将`wwwroot`和`acmeout`两个文件夹的所有者修改为`root:root`
```bash
chown -R root:root bark-server/wwwroot
chown -R root:root bark-server/acmeout
```

### 确定是否添加环境变量
申请https证书要通过添加环境变量添加api key和secert，以下是添加环境变量过程，但这要看情况，如果你申请通配符证书，那么就需要添加，如果你不申请通配符证书，那可以不添加，而且不是所有域名都可以申请通配符证书，像.cf .ml .tk .ga .gq这些freenom的免费域名，是无法申请通配符证书的，这些域名就不需要添加环境变量了。

添加环境变量的话，需要看你域名是在哪个平台解析的，比如你可能是阿里云的，那要添加的就是Ali_Key和Ali_Secret，比如你可能是cloudflare的那就添加CF_Email和CF_Key，其它的请自己从[这里](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi)找到 对应文件，然后点进去看源码里用的哪个变量。

---

- 1、添加环境变量，默认是添加到.bashrc中，如果你用的是zsh，那么你要把命令中的.bashrc修改为.zshrc
```bash
cat > ~/.bashrc << EOF
export CF_Email=zhangsan@163.com
export CF_Key=sdf39sdDfsjDkJN#)EJD
EOF
```
注意：执行上边命令前，要把CF_Email和CF_Key修改为你自己的，而且未必是CF_Email和CF_Key，具体要看你是在哪个网站中解析你的域名的。
- 2、让环境变量生效(如果你用的是zsh，要把下边的.bashrc换成.zshrc)
```bash
source ~/.bashrc
```
- 3、确认环境变量确实已生效，如果生效，运行以下两个会输出你前面填到
```bash
echo $CF_Email
echo $CF_Key
```

### 启动环境
进入bark-server文件夹
```bash
cd bark-server
```

运行以下命令启动(要等一小会儿)
```bash
docker compose up -d
```

启动后运行以下命令查看进程
```bash
docker compose ps
```

输出结果如下的结果
```bash
NAME                COMMAND                  SERVICE             STATUS              PORTS
acme.sh             "/usr/sbin/crond -f …"   acme.sh             running
bark-server         "/entrypoint.sh bark…"   bark-server         running
nginx               "/docker-entrypoint.…"   nginx
```

进入acme.sh容器中
```bash
docker exec -it acme.sh sh
```










----

以下是申请证书，有两种方式：

- 1、dns_1方式，不能是.cf .ml .tk .ga .gq这些免费域名，并且需要添加环境变量(前面有说过)；
- 2、http_01方式，任何域名都可以，不用添加环境变量，但不能申请通配符证书(不推荐，除非你是免费域名)；

申请证书方式一：执行以下命令申请证书(执行前请把命令中的域名改成你自己的)
```bash
acme.sh --issue --dns dns_cf -d zhangsan.com -d '*.zhangsan.com' --keylength ec-256
```

申请证书方式二：执行以下命令申请证书(执行前请把命令中的域名改成你自己的)
```bash
acme.sh --issue -d bark.zhangsan.com -d bark-cdn.zhangsan.com --webroot /data/wwwroot/well-known/  --home /root/acmeout --keylength ec-256
```

申请证书成功后，安装证书
```bash
acme.sh --install-cert --ecc --home /root/acmeout -d 'bark.zhangsan.com' \
--key-file       /root/certs/private.pem  \
--fullchain-file /root/certs/fullchain.pem
```

安装成功是这样的
```bash
[Fri Nov 11 21:57:17 CST 2022] Installing key to: /root/certs/private.pem
[Fri Nov 11 21:57:17 CST 2022] Installing full chain to: /root/certs/fullchain.pem
```

申请成功后，使用`exit`命令退出容器，返回到宿主机中。

### 启动nginx的https功能
- 1、进入bark-server/conf/nginx/vhost/文件夹，复制一份bark.example.com.conf.bak，并重命名为：bark.zhangsan.com.conf，并把该文件内部的所有域名都改为bark.zhangsan.com（注意bark.zhangsan.com是你自己的域名，如果你自己的域名不是这个，那你要改成你自己的）；
- 2、进入bark-server文件夹`cd bark-server`，然后运行`docker compose restart nginx`重启nginx；

### 测试是否成功
```bash
curl https://bark.zhangsan.com/ping
```

