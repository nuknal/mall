# 小程序商城以及闪销系统

- [技术栈](#技术栈)
- [架构](#架构)
- [代码结构](#代码结构)
  - [基础包 shanlaan/mall](#基础包：shanlaan/mall)
  - [服务模板 service_tpl](#服务模板service_tpl)
  - [闪销门店](#闪销门店)
  - [数据可视化](#数据可视化)
- [部署](#部署)
  - [云服务器](#云服务器)
  - [基础组件](#基础组件)
    - [Nginx](#nginx)
    - [服务发现 Consul](#服务发现Consul)
    - [数据订阅组件 Canal](#数据订阅组件Canal)
    - [延时队列 Delay-Queue](#延时队列delay-queue)
    - [监控](#监控)
    - [日志转储 Fluentd](#日志转储fluentd)
    - [链路跟踪 Jaeger](#链路跟踪jaeger)
    - [数据库索引服务 Sonic](#数据库索引服务Sonic)
  - [微信接口中控服务](#微信接口中控服务)
  - [商城服务 mall](#商城服务mall)
  - [闪销门店服务 store](#闪销门店服务store)

# 技术栈

- 主要语言：[Go](https://golang.org) v1.13：使用 go mod 做依赖管理
- RPC：[rpcx](https://github.com/smallnest/rpcx)，性能优秀，可拔插的插件化框架。
- 数据库：MySQL 5.7
- 缓存：Redis 4
- 消息队列：[RMQ](https://github.com/adjust/rmq)，基于 Redis，主要考虑的是成本和数据量小。支持 ack，可用性依赖 Redis
- 延迟队列：[delay-queue](https://github.com/ouqiang/delay-queue)，基于 Redis
- 任务调度：[Beanstalkd](https://github.com/beanstalkd/beanstalkd)
- 服务注册发现：[Consul](https://github.com/hashicorp/consul)，可集群可单机部署
- 监控：[Prometheus](https://github.com/prometheus/prometheus) + [Grafana](https://github.com/grafana/grafana)
- 链路跟踪：[Jaeger](https://github.com/jaegertracing/jaeger) + Cassandra，可替换，源码级接口基于 OpenTracing
- 配置中心：基于 Consul 的 kv 存储实现
- 搜索：[Sonic](https://github.com/valeriansaliou/sonic)，一个 Rust 实现的轻量级索引引擎，资源占用少，在简单应用场景下比 ElasticSearch 更适合
- 缓存数据增量同步：[Canal](https://github.com/alibaba/canal)，通过订阅 MySQL Binlog 实现
- GraphQL 网关：实现[GraphQL](https://graphql.org/)服务器
- 销售数据可视化：[Metabase](https://github.com/metabase/metabase)
- Web 服务器：[Nginx](https://nginx.org/)

# 架构

采用微服务架构

# 代码结构

$GOPATH/shanlaan/mall，采用**go1.13**不限制一定要放在 GOPATH 下

## Git 敏感文件加解密

使用**git-crypt 和 gpg**来加密含秘钥信息的文件: _.gitattributes_

```shell
# 中控服务器的配置文件
wechat-css/confing/*
# 其它文件, 如.env docker-compose.yml 中含数据库账号密码的也可放入
# 目前未加入是基于数据库只接受内网连接
```

使用方法:

```shell
# 安装工具
brew install git-crypt
brew install gpg
# 生成秘钥
gpg --gen-key   # 生成密钥（公钥和私钥），按照流程提示进行
gpg --list-keys # 会列出当前所有的密钥，检查刚才的密钥是否生成成功
# 加密
cd path/to/project
git-crypt init # 类似于git init，安装git-crypt到项目中
git-crypt add-gpg-user shanlaan # 添加密钥用户
vi .gitattributes
file/to/crypt filter=git-crypt diff=git-crypt
# 导出秘钥，项目成员(有权限)可共用
git-crypt export-key ~/Desktop/git-crypt-key
# 解密
git-crypt unlock /path/to/git-crypt-key
```

## 基础包 shanlaan/mall

```shell
├── alimq     # RocketMQ客户端实现，目前已不适用
├── analysis  # 埋点数据采集 未正式启用
├── assets    # 积分服务
├── auth      # 登录/鉴权
├── beanstalk # beanstalkd客户端，目前已不适用
├── cache     # 缓存同步服务
│   ├── cmd
│   └── pkg
│       ├── cache  # 缓存同步逻辑
│       ├── canald # 未启用（替换阿里canal的实现）
│       ├── canalx # 阿里Canal客户端，binlog消费者
│       └── client # 供其它服务调用的缓存读取基础接口
├── cart      # 购物车服务
├── common    # 通用接口
│   ├── alimq   # 队列相关消息体定义
│   ├── authx   # jwt
│   ├── cache   # 已不适用
│   ├── consts  # 常量定义
│   ├── cron    # 定时任务
│   ├── db      # gorm数据库连接以及模型代码生成工具
│   ├── decimal # 高精度浮点数运算，由于早期数据数据库设计使用的浮点数
│   ├── dqcli   # delay-queue客户端
│   ├── errors  # 错误
│   ├── gql     # graphql 例子，基于 gopher-graph
│   ├── idgen   # id生成，基于redis实现的自增id
│   ├── img     # 阿里云oss图片上传接口
│   ├── kvconf  # 配置中心实现
│   ├── lock    # 分布式锁
│   ├── pprof   # go运行时性能监控
│   ├── pub     # beanstalkd 任务订阅
│   ├── redips  # 基于redis实现的pub-sub
│   ├── redisx  # redis连接接口封装
│   ├── rmq     # 基于redis的消息队列 fork
│   ├── rmqx    # rmq队列消息生成者和消费者
│   ├── rpcxx   # rpcx的封装，主要添加链路跟踪、指标上报接口
│   ├── service_tpl # 服务代码接口模板，实现clean架构
│   ├── snowflake # 唯一ID生成器
│   ├── sonicx    # sonic 搜索客户端，sonic服务器ip读取环境变量SONIC_HOST
│   ├── sonyflake
│   ├── tracer  # 链路跟踪：jaeger客户端连接
│   ├── validator # 手机号等数据校验
│   ├── version # 编译版本
│   └── wx      # 微信接口
├── config
├── config-server # 配置中心，基于consul实现
├── coupon       # 优惠券服务
├── delay-queue  # 延时队列 fork
├── deploy       # 部署文件缓存目录
├── distribution # 分销服务
├── go-mysql-cache # 不依赖阿里canal的缓存同步服务，未启用
├── go-mysql-sonic # mysql搜索索引建立服务, 索引结果id:3455
├── goods        # 商品服务
├── graphql-gateway # graphql网关实现
├── groupon      # 团购 未实现
├── home         # 小程序主页
├── kingdee      # 金蝶ERP订单同步接口
├── membership   # 会员服务
├── message      # 消息服务，未实现。定位是短信、微信公众号、服务号消息统一发送服务
├── order        # 订单服务
├── pay          # 微信支付服务
├── price        # 价格服务 未实现
├── promotion    # 营销活动服务
├── risk-control # 风控，未实现
├── seckill      # 秒杀，未实现
├── stock        # 库存服务
├── store        # 闪销系统门店
├── support      # 基础支撑服务
├── user         # 用户服务
├── vcard        # 储值卡服务
├── vendor       # 依赖
├── vlog         # 访问数据，已不适用
└── wechat-ccs   # 微信token、js-ticket统一获取服务
├── .env      # 配置相关环境变量
├── make.sh   # 部署单个服务 ./make.sh account dev
├── deploy.sh # 全部服务部署脚本 接收一个参数输入dev|prod
├── docker-compose.yml.dev  # mall开发环境部署文件
├── docker-compose.yml.prod # mall生产环境部署文件
```

## 服务模板 service_tpl

遵循 **_Clean Architecture_** 代码架构

https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

```shell
├── Dockerfile  # docker
├── Makefile    # 编译
├── cmd
│   └── main.go # 入口
└── pkg         # 实现
    ├── errors  # 错误定义
    │   └── errors.go
    ├── repository # 数据访问
    │   ├── cache.go
    │   └── repo.go
    ├── repository.go
    ├── service    # 对外服务接口实现
    │   └── service.go
    ├── service.go # 对外服务接口声明
    ├── usecase    # 业务实例
    │   └── usecase.go
    ├── usecase.go
    └── util
```

## GraphQL 网关

```shell
├── Dockerfile
├── Makefile
├── consts
├── gapi    # graphql-gen-go 生成代码
├── gapix   # gapi类型扩展
├── gateway # 网关实现，使用filter对request和response进行过滤
├── parser  # graphql 请求解析
├── resolver # schema对应的resolver，调用后端服务
├── schema   # schema 定义
│   ├── bindata.sh
│   ├── mutation.graphql
│   ├── query.graphql
│   ├── schema.graphql
│   └── type
└── tool
    ├── doc     # 生成文档
    ├── gapi    # gen.sh 生成文件临时目录
    ├── gen.sh  # 生成脚本 `cd tool && ./gen.sh`
    ├── goreplay
    │   └── goreplay.md
    ├── graphql-gen-go # 从schema.graphql生成.go文件
    ├── schema.all.graphql
    └── wrk   # 压力测试
        ├── graphql.lua
        └── wrk.sh
```

## 闪销门店

```shell
├── account   # 门店管理员账号
├── common    # lib
├── deploy    # 部署文件
├── deploy.sh # 全部服务部署脚本 接收一个参数输入dev|prod
├── docker-compose.yml
├── docker-compose.yml.dev  # 开发环境部署文件
├── docker-compose.yml.prod # 生产环境部署文件
├── gateway   # 网关
├── .env      # 配置相关环境变量
├── make.sh   # 部署单个服务 ./make.sh account dev
├── message   # 消息服务
├── order     # 门店订单服务
└── store     # 门店服务
```

## 数据可视化

- 没有数据，未实现。后期可使用 Metabase

# 部署

Docker 部署，配置通过.env 写入 docker 实例环境变量中

## 宿主机环境变量

小程序秘钥等敏感信息配置，Docker 通过读取宿主机对应的环境变量获取

```shell
# 宿主机IP
export DOCKERHOST=$(ip -o -4  address show  | awk ' NR==2 { gsub(/\/.*/, "", $4); print $4 } ')
# 微信小程序
export WXMP_APPID=""
export WXMP_APPSECRET=""
# 微信公众号
export WXOA_APPID=""
export WXOA_APPSECRET=""
# JWT签名
export JWT_SIGNING_KEY=""
# 支付
export WXMP_PAYMENT_SECRET_KEY=""
```

## 云服务器

- 阿里云 ECS 生产环境 IP: -，通过 ssh key 连接，运行根目录：/data
- MySQL：阿里云
- Redis：阿里云

## 基础组件

```shell
scp -r shanlaan/mall/support/consul root@120.79.36.24:data/mall/support/consul
ssh root@ip
```

以 docker 部署基础支撑环境，以下部署命令如未指出在**本地源码目录**执行，则为在服务器运行

### Nginx

直接安装在服务器内，配置文件/etc/nginx

### 服务发现 Consul

```shell
cd data/mall/support/consul
docker-compose up -d
```

**服务注册中心可采用云 Redis 实现，因为对于独立运维 Consul 而言，云端的 Redis 可用性显然要高**

### 数据订阅组件 Canal

`rpc-cache`服务使用该组件订阅`MySQL Binlog`同步到`redis`，具体使用方式参考：

https://github.com/alibaba/canal
目录：/data/canal

配置文件：shanlaan/mall/support/canal/conf.prod

### 延时队列 Delay-Queue

基于 Redis 实现的延时队列，进入**本地源码目录**执行部署脚本

```shell
cd shanlaan/mall/delay-queue
./deploy.sh dev   # dev 部署到开发环境; prod 部署到生产环境
```

### 监控

```shell
cd /data/mall/support/dockprom
docker-compose -f docker-compose.yml up -d          # 启动Prometheus和Grafana服务器
docker-compose -f docker-compose.exporter.yml up -d # 启动Prometheus的Exporter
```

### 日志转储 Fluentd

所有**Docker**日志通过`logging-driver: fluentd` 导出到**Fluentd**，目前所有日志仅转储到文件 **/data/log**

**Fluentd**的配置问文件位于 `shanlaan/mall/support/fluentd/etc`

```shell
cd /data/mall/support/fluentd
docker-compose up -d
```

### 链路跟踪 Jaeger

使用 Jaeger 作为链路跟踪服务器，在后端程序通过 Opentracing 接口上报链路调用数据

```shell
cd /data/mall/support/jaeger
docker-compose up -d
```

### 数据库索引服务 Sonic

- 使用 Sonic 建立 text->idObject 是索引

```shell
cd /data/mall/support/sonic
docker-compose up -d
```

- 数据同步服务 go-mysql-sonic

```shell
# 本地目录
cd shalaan/mall/go-mysql-sonic
./deploy.sh dev|prod
# 配置文件目录etc river.toml 配置服务参数，索引的mysql 表
```

## 微信接口中控服务

负责定时向微信服务器获取 AccessToken 和 JsTicket

进入**本地源码目录 shanlaan/mall/wechat-ccs**执行脚本部署

```shell
cd shanlaan/mall/wechat-ccs
./deploy.sh dev   # dev 部署到开发环境; prod 部署到生产环境
```

## 商城服务 mall

进入**本地源码目录 shanlaan/mall**执行脚本部署

```shell
cd shanlaan/mall
./deploy.sh dev   # dev 部署到开发环境; prod 部署到生产环境
```

## 闪销门店服务 store

进入**本地源码目录 shanlaan/mall/store**执行脚本部署

```shell
cd shanlaan/mall/store
./deploy.sh dev   # dev 部署到开发环境; prod 部署到生产环境
```
