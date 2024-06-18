---
title: 基于docker(docker-compose)的 mongodb初始化导入json文件数据
categories: [编程, 中间件 ]
tags: [docker,docker-compose,mongodb]
---

最近在被割韭菜，遇到了一个问题，那就是如何在初始化mongodb容器的时候就把一些数据导入进去，并且是从一个json后缀的文件中导入。我翻遍国内平台，啥都解决不了，看了dockerhub中的mongo官方的用法也是含糊其辞。于是，我跑去stackoverflow和google搜了一下，然后又折腾了一会总算出来了。下面是两个相关的链接：
-  [Initialize MongoDB running on a Docker container](https://faun.pub/initialize-mongodb-running-on-a-docker-container-889a43c5668a)
- [https://time-is-life.fun/how-to-seed-data-into-mongo-with-docker](https://time-is-life.fun/how-to-seed-data-into-mongo-with-docker/)

参考上面两个链接基本就可以解决，不过我遇到了几个坑，所以记录一下

## 前提
[dockerhub mongo](https://hub.docker.com/_/mongo)

三个环境变量：
1. `MONGO_INITDB_ROOT_USERNAME`
2. `MONGO_INITDB_ROOT_PASSWORD`
3. `MONGO_INITDB_DATABASE`


前两个很好理解，而第三个来看官网的狡辩：

> This variable allows you to specify the name of a database to be used for creation scripts in /docker-entrypoint-initdb.d/\*.js (see Initializing a fresh instance below). MongoDB is fundamentally designed for "create on first use", so if you do not insert data with your JavaScript files, then no database is created.
也就是只有在/docker-entrypoint-initdb.d/\*.js文件中做插入操作才会生成这个数据库。（==mongodb可以执行/docker-entrypoint-initdb.d文件夹下.js文件和.sh文件==）
>
然后官网关于如何初始化数据库实例：
> Initializing a fresh instance
> When a container is started for the first time it will execute files with extensions .sh and .js that are found in /docker-entrypoint-initdb.d. ==Files will be executed in alphabetical order==. .js files will be executed by mongo using the database specified by the MONGO_INITDB_DATABASE variable, if it is present, or test otherwise. You may also switch databases within the .js script.

## 实践操作最终版
ok，看完两个链接和官网文档，我做了如下配置：
首先看看目录结构
![在这里插入图片描述](/assets/2022/07/25/1.png)

这里注意当有多个sh和js文件在/docker-entrypoint-initdb.d文件夹下时，是按照字母顺序执行的，所以，这里js要先于sh执行，先创建数据库和collection，在导入json数据。

`docker-compose.yml` 如下（这里没用`MONGO_INITDB_DATABASE`,是因为加进来sh文件里引用`$MONGO_INITDB_DATABASE`也会报错，暂时还没解决，可以自己试试，可以看看这儿：[How can I pass environment variables to mongo docker-entrypoint-initdb.d?](https://stackoverflow.com/questions/64606674/how-can-i-pass-environment-variables-to-mongo-docker-entrypoint-initdb-d)）：
```yaml
...
  jiefang-mongo:
    container_name: jiefang-mongo
    image: mongo
    build:
      context: ./mongo
      dockerfile: dockerfile
    ports:
      - "27017:27017"
    volumes:
      - ./mongo/conf/mongod.conf:/etc/mongod.conf
      - ./mongo/data:/data/db
      - ./mongo/init:/docker-entrypoint-initdb.d
...
```
mongo的`dockerfile`如下（没什么作用，主要缓存原始镜像，如果在compose里直接做镜像，每次改动还要重新拉取原始镜像）：
```yaml
FROM mongo
MAINTAINER bty
```

`01-init.js`文件如下：
```js
// Create user
dbAdmin = db.getSiblingDB("admin");
dbAdmin.createUser({
  user: "mongo",
  // 希望有机会值10个btc
  pwd: "10btc",
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }],
  mechanisms: ["SCRAM-SHA-1"],
});

// Authenticate user
dbAdmin.auth({
  user: "mongo",
  pwd: "10btc",
  mechanisms: ["SCRAM-SHA-1"],
  digestPassword: true,
});

// Create DB and collection
db = new Mongo().getDB("jiefangProj");
db.createCollection("t_xcp", { capped: false });
```
`02-init.sh`如下：
```bash
echo "########### Loading data to Mongo DB ###########"
mongoimport --jsonArray --db jiefangProj --file /docker-entrypoint-initdb.d/data.json --collection t_xcp
echo "########### Mongo DB data Loaded ###########"
```

## 坑点

这个`02-init.sh`可把我坑惨了，最开始是：
```bash
echo "########### Loading data to Mongo DB ###########"

mongoimport --jsonArray --db jiefangProj --collection t_xcp --file /docker-entrypoint-initdb.d/data.json 
echo "########### Mongo DB data Loaded ###########"
```
然后报错，说`\n`不是命令，就是什么空格、换行导致的，然后我把sh文件变紧凑了
```bash
echo "########### Loading data to Mongo DB ###########"
mongoimport --jsonArray --db jiefangProj --collection t_xcp --file /docker-entrypoint-initdb.d/data.json 
echo "########### Mongo DB data Loaded ###########"
```
然后一直报错：找不到这个文件或文件夹，搞了一下午。

我又把参数换了个位置：
```bash
echo "########### Loading data to Mongo DB ###########"
mongoimport --db jiefangProj --collection t_xcp --file /docker-entrypoint-initdb.d/data.json --jsonArray
echo "########### Mongo DB data Loaded ###########"
```
然后也报错，不过错误不同，说jsonArray不识别，我就用我聪明的大脑想到了，这个sh文件格式是不是有问题，但是搞了半天还是没找到原因。
最后吃完晚饭，又随便换了个参数位置：
```bash
echo "########### Loading data to Mongo DB ###########"
mongoimport --jsonArray --db jiefangProj --file /docker-entrypoint-initdb.d/data.json --collection t_xcp
echo "########### Mongo DB data Loaded ###########"
```
居然可以了！
----------------
后记，其实还是不行，最后不得以把两个echo指令删除，只保留哪个mongoimport，这真的没问题了。

-------------------------
过了几天，我发现数据安全有问题，重新改善以下：
## 数据被盗
发现mongodb数据老是被删除，然后还有一个新的collection显示的消息，这人英文不太好，太多病句：

> All your data is a backed up. You must pay 0.01 BTC to 12nx7Q6FAczH6yfhaEMJRmvRkavDTcwUJK 48 hours for recover it. After 48 hours expiration we will leaked and exposed all your data. In case of refusal to pay, we will contact the General Data Protection Regulation, GDPR and notify them that you store user data in an open form and is not safe. Under the rules of the law, you face a heavy fine or arrest and your base dump will be dropped from our server! You can buy bitcoin here, does not take much time to buy https://localbitcoins.com with this guide https://localbitcoins.com/guides/how-to-buy-bitcoins After paying write to me in the mail with your DB IP:
georgefloyd666@cock.li
yourdad4@cock.li

我也是醉了，发现之前设置的密码什么的都没用，可以直接登陆，原因是验证登陆没有开，开了之后，`mongod.conf`文件如下：
```yaml
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /data/db
  journal:
    enabled: true
#  engine:
#  wiredTiger:

# where to write logging data.
systemLog:
  logAppend: true
  destination: file
  logRotate: reopen
  path: "/var/log/mongodb/mongod.log"
# network interfaces
net:
  port: 27017
  # 0.0.0.0就是可以远程登陆，不限制bind
  bindIp: 0.0.0.0


# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  # 这里，验证登陆要打开
  authorization: enabled
# 慢查询日志
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  

```
然后捣鼓几次发现上面的初始化数据库和创建用户的方式总是有权限问题，比如查看不了jiefangProj数据库内容什么的。最后做了参考了StackOverflow的一篇问答解决了。
[How to create a DB for MongoDB container on start up?](https://stackoverflow.com/questions/42912755/how-to-create-a-db-for-mongodb-container-on-start-up/42917632)

那么我把我的整个需求整理下：
1. 为mongodb创建ROOT用户
2. 创建一个新的数据库jiefangproj
3. 为jiefangproj数据库创建单独的用户
4. 把json文件导入jiefangproj的t_xcp的collection中

OK，下面是我的操作

`docker-compose.yml`如下：
```yml
version : '3.8'
services:
  jiefang-mongo:
    container_name: jiefang-mongo
    image: mongo
    build:
      context: ./mongo
      dockerfile: dockerfile
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: rootpwd
      MONGO_INITDB_DATABASE: jiefangproj # js文件的操作会在该数据库下完成
    volumes:
      - ./mongo/conf:/etc/mongo # 配置文件
      - ./mongo/log:/var/log/mongodb # 日志目录挂载
      - ./mongo/data:/data/db # 数据目录挂载
      - ./mongo/init:/docker-entrypoint-initdb.d # 初始化操作
    command: --config /etc/mongo/mongod.conf # 指定上面的配置文件运行
```

其中init文件夹内的文件如上面的那个图所示。

`js`文件如下
```javascript
db.createUser({
  user: "username",
  pwd: "password",
  // 这个role有很多内置的角色
  // 具体参考https://www.mongodb.com/docs/manual/core/authorization/
  roles: [{ role: "dbOwner", db: "jiefangproj" }],
  mechanisms: ["SCRAM-SHA-1"],
});
```
`sh`文件如下：
```bash
mongoimport --jsonArray --db jiefangproj  --collection t_xcp --file /docker-entrypoint-initdb.d/data.json
```
