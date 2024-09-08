# 背景

种子网站由于版权的因素，成为了各国重点打击的对象，经常性被和谐，比如大名鼎鼎的海盗湾，另外不同的种子的网站资源质量也是参差不齐，资源不一定都能搜索到。用别人的资源站，有很多需求无法满足，与其依赖别人，不如我们自己动手搭建一套种子站点，顺便也带大家熟悉一下这其中的原理。

# 搭建过程

## 购买一台vps主机

优先选择国外的vps主机，国内主机的端口有可能会被封了。国外vps厂商很多，随便选就可以，配置不用太高，正常 2核2g+独立ip就够用了。我自己是在[zgovps](https://clients.zgovps.com/) 买了一台 ,配置如下

![vps](./img/vps.png)

购买完，要安装一下操作系统，系统我推荐你选择用ubuntu，当然其他的linux发行版本也可以选择

## 部署应用

我这里以ubuntu的系统为例子演示部署过程

1. 安装docker

```Plain
sudo snap install docker
```

2. 创建docker-compose.yml 填入下面内容

```Bash
version: "3"

services:
  bitmagnet-next-web:
    image: journey0ad/bitmagnet-next-web:latest
    container_name: bitmagnet-next-web
    ports:
      - "3000:3000"
    restart: unless-stopped
    environment:
      - POSTGRES_DB_URL=postgres://postgres:T0vTZCWWFtxmfCV5@postgres:5432/bitmagnet
    depends_on:
      postgres:
        condition: service_healthy

  bitmagnet:
    image: ghcr.io/bitmagnet-io/bitmagnet:latest
    container_name: bitmagnet
    ports:
      # API and WebUI port:
      - "127.0.0.1:3333:3333"
      # BitTorrent ports:
      - "3334:3334/tcp"
      - "3334:3334/udp"
    restart: unless-stopped
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_PASSWORD=T0vTZCWWFtxmfCV5
      # - TMDB_API_KEY=your_api_key
    command:
      - worker
      - run
      - --keys=http_server
      - --keys=queue_server
      # disable the next line to run without DHT crawler
      - --keys=dht_crawler
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    container_name: bitmagnet-postgres
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:42401:5432"
    restart: unless-stopped
    environment:
      - POSTGRES_PASSWORD=T0vTZCWWFtxmfCV5
      - POSTGRES_DB=bitmagnet
      - PGUSER=postgres
    shm_size: 1g
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready
      start_period: 20s
      interval: 10s
  nginx:
    image: nginx:latest
    container_name: bitmagnet-nginx
    ports:
      - "4444:4444"
    volumes:
      - ./default.conf:/etc/nginx/conf.d/default.conf
    command: >
      /bin/sh -c "apt-get update && apt-get install -y apache2-utils && 
      htpasswd -cb /etc/nginx/.htpasswd test abc1234567890 && 
      nginx -g 'daemon off;'"
    depends_on:
      - bitmagnet
    restart: unless-stopped
```

3. 创建default.conf填入下面内容

```Bash
server {
    listen 4444;
    server_name localhost;

    location / {
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://bitmagnet:3333;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Protocol $scheme;
    }
}
```

4. 启动服务

```Bash
docker-compose up -d
```

5. 防火墙开放3000，4444，3334端口 

6. 验证是否正常

- 访问 http://主机ip:3000

  ![serarch1](./img/search1.png)

- 访问   http://主机ip:4444

  - 账号密码：test  abc1234567890

  ![serarch1](./img/search2.png)

# 种子搜索站点原理分析

![yl](./img/yl.png)
