---
layout: post
title: "kong插件"
subtitle: 'kong插件开发指南'
author: "odin"
header-style: text
tags:
  - 中间件
  - kong
---
![]({{site.baseurl}}/img/in-post/post-kong/logo-color.png)

记录docker下快速部署kong的步骤。

## 1.kong部署
### 1.1）创建kong容器网络
首先创建kong-net容器网络。默认设置为`bridge`
```shell
docker network create kong-net
```

部署`postgresql`,kong默认使用postgres做持久化。
```
docker run -d --name kong-database \
--network=kong-net \
-p 5432:5432 \
-e "POSTGRES_USER=kong" \
-e "POSTGRES_DB=kong" \
-e "POSTGRES_PASSWORD=123456" \
-v /home/java_user/kong/data/postgres:/var/lib/postgresql/data \
postgres:9.6
```

> --network: 指定docker网络  
> POSTGRES_USER，POSTGRES_DB，POSTGRES_PASSWORD：指定pg的数据库，用户和密码
> -v：挂在数据卷

### 1.2）短暂启动kong的数据库初始化镜像
```
docker run --rm \
--network=kong-net \
-e "KONG_DATABASE=postgres" \
-e "KONG_PG_HOST=kong-database" \
-e "KONG_PG_PASSWORD=123456" \
-e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
kong kong migrations bootstrap
```

> KONG_DATABASE,KONG_PG_HOST,KONG_PG_PASSWORD：指定数据库类型，host和密码。由于network与数据库为同一容器下，故可以使用容器名访问。

### 1.3）启动kong
```
docker run -d --name kong \
--network=kong-net \
-e "KONG_DATABASE=postgres" \
-e "KONG_PG_HOST=kong-database" \
-e "KONG_PG_PASSWORD=123456" \
-e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
-e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
-e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
-e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
-e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
-e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
-v /home/kong/data/log/kong:/var/log \
-p 800:8000 \
-p 443:8443 \
-p 8001:8001 \
-p 8444:8444 \
kong
```

> KONG_DATABASE,KONG_PG_HOST,KONG_PG_PASSWORD：同上，不再赘述。
> KONG_PROXY_ACCESS_LOG：指定代理访问日志。
> KONG_ADMIN_ACCESS_LOG：指定admin接口日志。
> KONG_PROXY_ERROR_LOG：指定代理错误日志。
> KONG_ADMIN_LISTEN：kong管理api的http和https端口。
> -v：挂载数据卷。

基本和kong服务默认的配置参数一致，只需要加上KONG_前缀。

## 2.konga
konga市非官方gui，但是比ashboard功能丰富页面美观，是目前开源项目中最好的选择。
### 2.1）初始化konga相关数据
主要是账户和一些基本设置信息保存。
```
docker run --rm \
    --network=kong-net \
    pantsel/konga -c prepare -a postgres -u postgresql://kong:123456@kong-database:5432/konga_db
```

数据库url格式 postgresql://用户名:数据库密码@kong-database:端口/库名

### 2.2）启动konga
```
docker run -p 1337:1337 --name konga  --network=kong-net \
             -e "DB_ADAPTER=postgres" \
             -e "DB_HOST=kong-database" \
             -e "DB_PORT=5432" \
             -e "DB_USER=kong" \
             -e "DB_PASSWORD=123456" \
             -e "DB_DATABASE=konga_db" \
             -e "KONGA_LOG_LEVEL=debug" \
             -e "NODE_ENV=production" \
             pantsel/konga
```

> DB_ADAPTER：指定数据库类型。
> DB_HOST,DB_PORT,DB_USER,DB_PASSWORD,DB_DATABASE：指定数据库host，端口，用户，密码，库名。
> KONGA_LOG_LEVEL：设置日志级别。
> NODE_ENV：设置环境配置。

### 2.3）注册konga
![]({{site.baseurl}}/img/in-post/post-kong/kong-regist.png)

### 2.4）konga连接配置
name：随意。
kong admin url：http://host:port

本地容器部署时，kong和konga不在同一个容器内，kong admin url的kong地址记得用ip地址，不要用127.0.0.1