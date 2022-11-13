# bark-server-docker使用说明

**原理**：通过nginx承接https(443端口)流量，并转发给bark-server的8080端口，以达到加密的目的。而acme.sh用于申请证书，当你用http-01方式来申请证书时，需要nginx开一个80端口来让letsencrypt校验，但由于还没有申请到证书，所以443端口那个配置文件暂时不能启用，等申请到证书，安装到目的文件夹后，再启用443端口那个nginx配置文件(复制一份并去掉`.bak`后缀就会启用)。

## 环境准备
### 工具材料
- 1、准备一台Linux服务器（最好是国外服务器，这样不用备案更省事，这里以Debian 11系统为例），假设ip为：12.34.56.78；
- 2、准备一个域名，假设为：zhangsan.com；
- 3、解析一个三级域名到到你的服务器，假设为：bark.zhangsan.com，添加一条A记录，解析到你的服务器：12.34.56.78，并且`ping bark.zhangsan.com`能ping通（当然你也可以解析多个域名到同一个服务器，比如如果是使用cloudflare解析，我一般会再解析一个`bark-cdn.zhangsan.com`到我的服务器中，这个要开启小云朵表示走cdn）；

### 安装docker
在服务器中安装docker，请参考官方文档：[Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/)。

### 下载bark-server-docker
我准备了一个bark-server的docker运行环境，直接clone下来就行：
```bash
cd /root/
git clone https://github.com/xiebruce/bark-server-docker.git
```

## 申请https证书
申请https证书有以下两种方式(两种方式选一种就行，不要两种都用)：

- 1、dns-01方式：不能是.cf .ml .tk .ga .gq这些免费域名，并且需要添加环境变量(前面有说过)；
- 2、http-01方式：任何域名都可以，不用添加环境变量，但不能申请通配符证书(不推荐，除非你是免费域名)；

**建议**：如果你的域名不是.cf .ml .tk .ga .gq这些免费域名，建议使用第一种方式。

### 修改nginx配置文件
dns-01方式不用先修改nginx配置文件，而http-01方式需要修改nginx配置文件(80端口那个)，为了统一，这里先统一修改一下80端口的nginx配置文件。

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

打开`bark.zhangsan.com-80.conf`，把里面的`example.com`修改成`zhangsan.com`(注意要换成你自己的域名)，其实主要就是`server_name`,`access_log`和`error_log`这三个的值，如下所示：
```bash
server_name bark.zhangsan.com bark-cdn.zhangsan.com;

# access_log /data/wwwlogs/bark.zhangsan.com_nginx.access.log combined;
access_log /data/wwwlogs/bark.zhangsan.com_nginx.access.log combined buffer=1k;
error_log /data/wwwlogs/bark.zhangsan.com_nginx.error.log error;
```

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

如果添加错了，请自己用vi/vim/nano命令去`~/.bashrc`文件或`~/.zshrc`文件中修改

---

让环境变量生效(如果你用的是zsh，要把下边的`.bashrc`换成`.zshrc`)
```bash
source ~/.bashrc
```

确认环境变量确实已生效，如果生效，运行以下两个命令，会分别输出你前面添加到环境变量中的值
```bash
echo $CF_Email
echo $CF_Key
```

启动所有容器
```bash
# 进入docker-compose.yml所在目录
cd bark-server-docker

# 启动所有容器
docker compose up -d

# 查看容器是否成功启动
docker ps
```

如果容器成功启动，显示应该类似如下
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

在容器中执行以下命令获取证书(别急着复制粘贴，要改参数，往下看)
```bash
acme.sh --issue --dns dns_cf -d zhangsan.com -d '*.zhangsan.com' --keylength ec-256
```
- `--dns dns_cf`表示你的域名是在cloudflare解析的，如果你是阿里云解析的，那就是`--dns dns_ali`，如果是其它服务商解析的，自己从[这里](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi)找，注意这个要与前面的环境变量对应；
- 命令中两处`zhangsan.com`要换成你自己的域名；

申请成功后，安装证书(`zhangsan.com`还是要改成你自己的域名)
```bash
acme.sh --install-cert --ecc --home /root/acmeout -d 'zhangsan.com' \
--key-file       /root/certs/private.pem  \
--fullchain-file /root/certs/fullchain.pem \
--reloadcmd      "docker restart nginx"
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
--fullchain-file /root/certs/fullchain.pem \
--reloadcmd      "docker restart nginx"
```

安装成功是这样的
```bash
[Fri Nov 11 21:57:17 CST 2022] Installing key to: /root/certs/private.pem
[Fri Nov 11 21:57:17 CST 2022] Installing full chain to: /root/certs/fullchain.pem
```

### 自动更新证书
acme.sh容器内部运行了一个crond服务，你在acme.sh容器内执行`crontab -e`可以看到有以下这行，表示每天凌晨3点07分会检查一次更新
```bash
7 3 * * * /root/.acme.sh/acme.sh --cron --home /root/acmeout
```

## 运行所有服务
### 再修改nginx配置文件
进入以下目录
```bash
cd bark-server-docker/conf/nginx/vhost
```

可以看到目录中有这三个文件
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
bark.zhangsan.com-80.conf
```

`bark.example.com-80.conf.bak`前面我们已经复制过一份了，现在只要把`bark.example.com.conf.bak`也复制一份，并改成类似如下的名称(其中zhangsan.com要改成你自己真实的域名)
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
bark.zhangsan.com-80.conf
bark.zhangsan.com.conf
```

与`bark.zhangsan.com-80.conf`一样，把`bark.zhangsan.com.conf`中的`example.com`改成`zhangsan.com`(当然你要用你真实的域名)，其实主要就是`server_name`,`access_log`和`error_log`这三个
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

### 测试ping
运行以下命令进行测试
```bash
curl https://bark.zhangsan.com/ping
```

如果一切正常，应该返回如下所示的json
```json
{"code":200,"message":"pong","timestamp":1668248534}
```

### 添加地址到Bark app
在Bark app中，点击底部第一个按钮“服务器” → 点击“右上角+号” → 把地址`https://bark.zhangsan.com`粘贴进去 → 点击右上角的对勾✓ → 它会返回到服务器列表中，至此就添加完成了。

### 如果之前已有nginx
如果你之前就已经有nginx，也配置好了证书，那么你可以把`docker-compose.yml`中的acme.sh模块(即以下模块)注释掉
```bash
# acme.sh，用于申请https证书
# 文档：https://github.com/acmesh-official/acme.sh/wiki/Run-acme.sh-in-docker
acme.sh:
build:
  context: .
  dockerfile: acme.sh-dockerfile
# 这个image就是上面构建的img，这样可以自定义名称
image: bruce/acme.sh
container_name: acme.sh
environment:
  - CF_Email=${CF_Email}
  - CF_Key=${CF_Key}
  # acme.sh工作目录，获取证书或日志文件都会向该目录写入，该参数也可在运行时通过--home来覆盖
  - LE_WORKING_DIR=/root/acmeout
  - TZ=Asia/Shanghai
volumes:
  - ./certs/:/root/certs/
  - ./wwwroot/:/data/wwwroot/
  - ./acmeout/:/root/acmeout/
network_mode: host
restart: always
```

然后在`server_name`模块中添加你要用于bark-server的域名
```nginx
server_name example.com www.example.com bark.zhangsan.com bark-cdn.zhangsan.com;
```

然后参考以下的写法，通过if来判断域名，相当于一个location两个用途，当符合if条件的时候，就用于bark-server，当不符合if条件的时候，就用于它之前的用途，当然location后不能有其它，只能有一个斜杠`/`
```nginx
location / {
    set $flag 0;
    #return 502 $host;
    if ( $host = "bark.example.com" ) {
        set $flag 1;
    }
    if ( $host = "bark-cdn.example.com" ) {
        set $flag 1;
    }
    if ( $flag = 0 ) {
        # 当请求的不是bark.example.com或bark-cdn.example.com时，做你想做的事
        return 302 https://www.xiebruce.top;
    }
    log_not_found on;
    # Replace http://192.168.1.123:8080 with the listening address of the bark server.
    proxy_pass http://127.0.0.1:8080;

    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect off;

    proxy_set_header Host              $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP         $remote_addr;
}
```