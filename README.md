# bark-server-docker使用说明

搭建bark-server主要是为了实现：[自动转发安卓机短信到iPhone](https://www.xiebruce.top/1826.html)。

---

本项目集成了四个容器：

- 1、bark-server：自建bark app服务器端；
- 2、chanify：自建chanify app服务器端(chanify app是与bark app类似的iPhone消息推送工具)；
- 3、nginx：由于bark-server和chanify都只支持http，不支持https，所以需要nginx来支持https，并把请求反代到bark-server和chanify；
- 4、acme.sh：nginx支持https需要申请证书，acme.sh就是用于申请证书以及自动续期证书的工具；

---

**原理**：通过nginx承接https(443端口)流量，并转发给bark-server的8080以及chanify的8081端口，以达到加密的目的。

而acme.sh用于申请证书，当你用http-01方式来申请证书时，需要nginx开一个80端口来让letsencrypt校验，但由于还没有申请到证书，所以443端口那个配置文件暂时不能启用，等申请到证书，安装到目的文件夹后，再启用443端口那个nginx配置文件(复制一份并去掉`.bak`后缀就会启用)，当然用dns-01方式申请就没这个问题，但有些域名又无法使用dns-01方式申请，比如freenom的免费域名使用cloudflare解析时就无法用dns-01方式申请(cloudflare禁止这类免费域名通过dns-01方式申请证书)。

## 环境准备
### 工具材料
- 1、准备一台Linux服务器（最好是国外服务器，这样不用备案更省事，这里以Debian 11系统为例），假设ip为：12.34.56.78；
- 2、准备一个域名，假设为：zhangsan.com；
- 3、解析两个三级域名到你的服务器ip(12.34.56.78)，假设分别为：bark.zhangsan.com、chanify.zhangsan.com，这两个域名分别用于bark-server和chanify，如果你只用其中一个，那就解析一个就行；

### 安装docker
在服务器中安装docker，请参考官方文档：[Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/)。

### 下载bark-server-docker
下载我准备好的docker-compose环境：
```bash
cd /root/
git clone https://github.com/xiebruce/bark-server-docker.git
```

clone下来的bark-server-docker文件夹中有一个docker-compose.yml文件，该文件是docker compose配置文件，它有四个模块，其中acme.sh和nginx肯定要用到的，而bark-server和chanify这两个一般人只要用其中一个就够了，你自己看情况，把不用的那个注释掉，当然如果你两个都用也行，它们是可以同时启动的。

比如你不想用chanify，那就把chanify模块注释掉
```yaml
# Chanify app服务器端: https://github.com/chanify/chanify
# 也是一个苹果消息推送服务器，但功能强大的多，但也比较复杂，用的人少
# chanify:
#   image: wizjin/chanify:dev
#   container_name: chanify
#   command: serve
#   environment:
#     - TZ=Asia/Shanghai
#   volumes:
#     - ./data/chanify:/data
#     - ./conf/chanify/chanify.yml:/root/.chanify.yml
#   network_mode: host
#   restart: always
```

## 申请https证书
申请https证书有以下两种方式(两种方式选一种就行，不要两种都用)：

- 1、dns-01方式：不能是.cf .ml .tk .ga .gq这些免费域名，并且需要添加环境变量(前面有说过)；
- 2、http-01方式：任何域名都可以，不用添加环境变量，但不能申请通配符证书(不推荐，除非你是免费域名)；

**建议**：如果你的域名不是.cf .ml .tk .ga .gq这些免费域名，建议使用第一种方式(即dns-01方式)。

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

把两个example配置文件分别复制一份，并修改文件名如下(其中zhangsan.com要改成你自己真实的域名)
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
bark.zhangsan.com-80.conf
bark.zhangsan.com.conf.bak
```
注意：`.conf`结尾表示启用该配置文件，这里先启用`bark.zhangsan.com-80.conf`(监听的80端口)，而`bark.zhangsan.com.conf.bak`暂时不启用，因为它监听的是443端口，需要tls证书，而现在还没有申请证书呢，所以不能启用它，否则会报错找不到证书。

然后分别把`bark.zhangsan.com-80.conf`和`bark.zhangsan.com.conf.bak`里面的`example.com`修改成`zhangsan.com`(注意要换成你自己的域名)，其实主要就是`server_name`,`access_log`和`error_log`这三个的值，如下所示：
```bash
server_name bark.zhangsan.com chanify.zhangsan.com;

# access_log /data/wwwlogs/zhangsan.com_nginx.access.log combined;
access_log /data/wwwlogs/zhangsan.com_nginx.access.log combined buffer=1k;
error_log /data/wwwlogs/zhangsan.com_nginx.error.log error;
```

### dns-01方式申请证书
执行以下命令添加环境变量(如果你用的是zsh，需要把`.bashrc`修改为`.zshrc`)，这里假设你是在cloudflare解析的域名
```bash
cat > ~/.bashrc << EOF
export CF_Email=zhangsan@163.com
export CF_Key=sdf39sdDfsjDkJN#)EJD
EOF
```
- CF_Email：就是你登录Cloudflare的邮箱；
- CF_Key：登录[Cloudflare](https://www.cloudflare.com/)后，点击[这个链接](https://dash.cloudflare.com/profile/api-tokens)，在里面找到“Global API Key”→点击它右侧的“View”按钮→输入登录Cloudflare的密码即可查看，请不要使用“Origin CA Key”，[acme.sh作者](https://github.com/Neilpang)在[这里](https://github.com/acmesh-official/acme.sh/issues/1976#issuecomment-450292853)有说Origin CA Key是不行的。

---

注意：执行上边命令前，要把CF_Email和CF_Key修改为你自己的，而且key未必是CF_Email和CF_Key，具体要看你是在哪个网站中解析你的域名的，比如如果是在阿里云解析域名，则需要变成
```bash
cat > ~/.bashrc << EOF
export Ali_Key=sdf39sdDfsjDkJN#)EJD
export Ali_Secret=sdf39sdDfsjDkJN#)EJD
EOF
```
**Ali_Key和Ali_Secret获取方式**：进入[阿里云控制台](https://home.console.aliyun.com/)→点击右上角用户名→accesskey管理→继续使用AccessKey→创建AccessKey→AccessKey ID对应上边的Ali_key，而Ali_Secret要点击“查看Secret”；

如果你的域名不是在Cloudflare解析，也不是阿里在云解析的，请自己从[这里](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi)找到对应文件，然后点进去看源码里用的哪个变量。

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

如果容器成功启动，显示应该类似如下(四个容器，本文开头有说)
```bash
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS     NAMES
1a96e50b4d49   wizjin/chanify:dev   "/usr/local/bin/chan…"   3 seconds ago   Up 2 seconds             chanify
bff0659b6f25   bruce/nginx          "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds             nginx
a566d5ca2c0f   bruce/acme.sh        "/usr/sbin/crond -f …"   3 seconds ago   Up 2 seconds             acme.sh
c56fc7cf6a25   finab/bark-server    "/entrypoint.sh bark…"   3 seconds ago   Up 2 seconds             bark-server
```

---

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
cd bark-server-docker
docker compose up -d
```

运行后，使用`docker ps`查看，应该是如下所示这样的
```bash
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS     NAMES
1a96e50b4d49   wizjin/chanify:dev   "/usr/local/bin/chan…"   3 seconds ago   Up 2 seconds             chanify
bff0659b6f25   bruce/nginx          "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds             nginx
a566d5ca2c0f   bruce/acme.sh        "/usr/sbin/crond -f …"   3 seconds ago   Up 2 seconds             acme.sh
c56fc7cf6a25   finab/bark-server    "/entrypoint.sh bark…"   3 seconds ago   Up 2 seconds             bark-server
```

先做一个访问测试，在wwwroot目录下创建目录和文件
```bash
mkdir -p wwwroot/well-known/.well-known/acme-challenge/
echo 'This is a test!' >> wwwroot/well-known/.well-known/acme-challenge/test.txt
```
然后在你电脑浏览器中访问：[http://bark.zhangsan.com/.well-known/acme-challenge/test.txt](http://bark.zhangsan.com/.well-known/acme-challenge/test.txt)

如果浏览器能正常输出“This is a test!”，说明访问是通的，现在你可以把刚刚添加的文件夹删掉了
```bash
rm -rf wwwroot/well-known/.well-known
```

**注**：做这个测试的原因，是因为acme.sh使用http-01方式申请letsencrypt证书时，letsencrypt服务器是会向上边那个测试链接发起一个GET请求的，只不过它不是请求test.txt，而是访问一个它自己创建的文件(文件名比较长)，所以我们必须保证这个链接能正常访问，否则申请证书肯定失败。

而我之所以在确认链接没问题后，又把`.well-known`文件夹删掉，是因为acme.sh在验证时会自动创建`.well-known/acme-challenge/`这个文件夹，以及自动在该文件夹下创建一个用户校验的文件(文件名格式类似这样“z0cTIz2zcORh47L0EwrSXKKoJxKMUTFK0EJIMrRWYaA”)，验证完它又会自动删掉，就好像一切都没发生过，所以不需要我们自己创建。

---

测试完上边的检验链接没问题后，我们进入acme.sh容器中
```bash
docker exec -it acme.sh sh
```

运行以下命令申请证书，当然`zhangsan.com`要换成你自己的真实域名(每个`-d`指定的域名都必须已经解析到当前服务器)
```bash
acme.sh --issue -d zhangsan.com -d www.zhangsan.com -d bark.zhangsan.com -d chanify.zhangsan.com --webroot /data/wwwroot/well-known/  --home /root/acmeout --keylength ec-256
```
**注意**：每个`-d`表示一个域名，如果你又解析了一个域名，你还得把它加到上边命令中，重新申请一次证书，不能申请通配符证书就是这么麻烦。

申请证书成功后，安装证书
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

### 自动更新证书
acme.sh容器内部运行了一个crond服务，你在acme.sh容器内执行`crontab -e`可以看到有以下这行，表示每天凌晨3点07分会检查一次更新
```bash
7 3 * * * /root/.acme.sh/acme.sh --cron --home /root/acmeout
```
注：虽然每天都会检查一次更新，但如果检查发现没有到期，它是不会更新证书的，如果到期了，它才会更新，而且更新完它会自动重启nginx服务器以新加载新证书(因为第一次安装证书时用`--reloadcmd`指定了安装证书后重启nginx，它是会记住这个命令的)。至于为什么在容器内它也能重启另一个容器，那是因为我在容器内安装了docker-cli(acme.sh-dockerfile自动安装的)，并且把`/var/run/docker.sock`映射进去了。

## 运行所有服务
### 修改chanify配置文件
配置如下，特别注意`endpoint`必须与你真实的域名(查看二维码的域名)一致，否则会导致无法添加节点
```yaml
server:
  # 只监听本机，外网通过nginx反代过来
  host: 127.0.0.1
  # 监听本机的8081端口，这个可以自己随便设置，只要没被占用就好
  port: 8081
  # 必须与你最终访问的url一致
  endpoint: https://chanify.zhangsan.com
  # 节点添加到手机后，会在手机节点列表中显示该名称
  name: "自定义名称"
  # 用于存储图片等文件(因为chanify不仅仅只支持推送文本)
  datapath: /data/
  http:
    # http读取超时时间
    - readtimeout: 10s
    # http写入超时时间
    - writetimeout: 10s
  register:
    # 设置为false理论上就无法添加该节点，但事实上还是可以添加，估计是bug
    enable: false
    whitelist: # whitelist for user register
      # 用户id，在Chanify app中的：设置→用户 里，这是白名单，添加后，即使enable为false，也能添加
      - ABLGYLOFFJSDFOISFHSIFDISDFWQ44YZ7VM
      # - <user id 2>
```

### 再修改nginx配置文件
进入以下目录
```bash
cd bark-server-docker/conf/nginx/vhost
```

可以看到目录中有这四个文件
```bash
bark.example.com-80.conf.bak
bark.example.com.conf.bak
bark.zhangsan.com-80.conf
bark.zhangsan.com.conf.bak
```

把`bark.zhangsan.com.conf.bak`后面的`.bak`去掉，表示启用该配置文件(`.conf`结尾就会被启用)，该配置文件为监听443端口的配置文件，因为前面已经申请了证书，所以这里可以启用它
```bash
mv bark.zhangsan.com.conf.bak bark.zhangsan.com.conf
```

检查80,443,8080,8081四个端口未被占用，因为80和443 nginx要使用，8080是bark-server的端口(无法改，代码中写死的，main.go文件中能搜到)，8081是chanify要占用
```bash
# grep 80，所有80开头的端口都会被列出
netstat -tlunp | grep 80

# grep 443，查找443端口是否被占用
netstat -tlunp | grep 443
```

如果上边说的四个端口都没被占用，那么就可以启动所有服务了
```bash
# 进入docker-compose.yml所在文件夹
cd bark-server-docker

# 启动所有服务
docker compose up -d
```

运行之后，可以看到四个docker容器都启动了，如下所示
```bash
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS     NAMES
1a96e50b4d49   wizjin/chanify:dev   "/usr/local/bin/chan…"   3 seconds ago   Up 2 seconds             chanify
bff0659b6f25   bruce/nginx          "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds             nginx
a566d5ca2c0f   bruce/acme.sh        "/usr/sbin/crond -f …"   3 seconds ago   Up 2 seconds             acme.sh
c56fc7cf6a25   finab/bark-server    "/entrypoint.sh bark…"   3 seconds ago   Up 2 seconds             bark-server
```

### 测试连通性
**Bark-server**：在浏览器中访问`https://bark.zhangsan.com/ping`，如果一切正常，应该显示如下所示的json
```json
{"code":200,"message":"pong","timestamp":1668248534}
```

至此，服务器就算是搭建完成了，你的消息推送地址为：`https://bark.zhangsan.com`，把它填到Bark app中就行。

---

**Chanify**：在浏览器中访问`https://chanify.zhangsan.com`，如果正常，会出来一个二维码，说明Chanify服务器运行正常。

### 添加地址到app中
**添加到Bark app中**：
在Bark app中，点击底部第一个按钮“服务器” → 点击“右上角+号” → 把地址`https://bark.zhangsan.com`粘贴进去 → 点击右上角的对勾✓ → 它会返回到服务器列表中，至此就添加完成了。

---

**添加地址到Chanify app中**：

- 1、打开[https://chanify.zhangsan.com](https://chanify.zhangsan.com)会出来二维码(注意要把zhangsan.com换成你自己的真实的域名)；
- 2、在Chanify app中，点击底部第二个按钮“节点” → 点击“右上角+号” → 扫第1步出来的码，会出来这节点信息；
- 3、点击第2步页面的最后的“添加节点”即可添加，如果你之前添加过，它会显示“更新节点”。

**注**：Chanify如果添加失败，一般是由于chanify配置文件中的endpoint与你访问二维码的链接不同导致的，把它修改为访问二维码的地址并重启(`docer restart chanify`)。

### 测试推送通知
**测试Bark推送**：
在浏览器中访问以下链接即可，其中`https://bark.example.com/<token>/`部分是可以从app服务器列表中直接复制的，`%0a`是换行符
```bash
https://bark.example.com/<token>/这是测试标题/这是测试内容%0a换行内容%0a再换行
```

如果推送成功，它会返回像以下这样的json，并且手机Bark app也能收到消息通知
```json
{"code":200,"message":"success","timestamp":1668536487}
```

---

**测试Chanify推送**：
在浏览器访问以下链接，在url中`%0a`表示换行，`<token>`要替换为chanify app中的真实token
```bash
https://chanify.zhangsan.com/v1/sender/<token>?sound=1&priority=10&title=hello&text=这是推送文本%0a试试换行%0a再换行%0a再换&copy=1234&autocopy=1
```
**token获取方式**：点击chanify app中的某个“频道”→右上角三个点→令牌(它默认有多个令牌，有自带的，有你自建的，选你自建的就行)。

如果正常，它会返回以下这样的json，并且手机Chanify也能收到消息通知
```json
{"request-uid":"4ba36d56-8069-4e20-a77a-40229b455453"}
```

### 如果之前已有nginx
如果你之前就已经有nginx，也配置好了证书，那么你可以把`docker-compose.yml`中的acme.sh模块和nginx模块都注释掉，然后在`server_name`模块中添加你要用于bark-server或chanify的域名
```nginx
server_name example.com www.example.com bark.example.com chanify.example.com;
```

然后参考以下的写法，通过if来判断域名以确认是否需要转发、转发到哪个端口等等，相当于一个location多个用途
```nginx
location / {
    log_not_found on;

    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect off;

    proxy_set_header Host              $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP         $remote_addr;

    # 特别注意：nginx配置文件的if是没有else的，虽然它能用&&和||符号，但不能用于表达式之间，而只能用于变量之间
    if ( $host = "example.com" ) {
        return 302 https://www.$host$request_uri;
        break;
    }

    # 通过域名来判断要转发到哪个端口，实现一个配置多用
    if ( $host = "bark.example.com" ) {
        proxy_pass http://127.0.0.1:8080;
        break;
    }

    if ( $host = "chanify.example.com" ) {
        proxy_pass http://127.0.0.1:8081;
        break;
    }

    # 默认反代地址（当前面if条件都不符合时，会走这里）
    try_files $uri $uri/ /index.html;
    # 如果本机没有搭建网站，也可以跳转到其它网站
    # return 302 https://www.xiebruce.top;
}
```