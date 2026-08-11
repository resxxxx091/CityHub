# CityHub 启动指南

## 1. 启动中间件

```bash
docker compose up -d
docker compose ps
```

首次启动 MySQL 会自动导入 `src/main/resources/db/hmdp.sql`。`mysql-data` 卷已经存在时不会再导，升级旧库要先执行 `src/main/resources/db/migrate-redis-stock-restored.sql`。

RocketMQ 只起了 NameServer 和 Broker，`autoCreateTopicEnable=true`，Topic 不用手动建。`docker/rocketmq/broker.conf` 里 `brokerIP1=127.0.0.1`，只适合本机开发。

## 2. 填配置

`src/main/resources/application.yaml` 里有三个占位符要改：MySQL 用户名、密码、Redis 地址。默认值和 compose 对得上，填 `root` / `root` / `127.0.0.1` 即可。

## 3. 启动后端

```bash
mvn spring-boot:run
```

或者打包后运行：

```bash
mvn clean package -DskipTests
java -jar target/hmdp-pro-0.0.1-SNAPSHOT.jar
```

默认端口 8081。

注意：Canal 默认关闭（`canal.enabled=false`），没有 Canal 实例也能正常启动；要开缓存驱逐链路，先起 Canal 服务再改成 `true`。

## 4. 切换秒杀写策略

`seckill.mode` 有两个值：

- `A`（默认）：入口 Lua 同步扣 Redis 库存并做一人一单，再用 RocketMQ 事务消息把「扣减成功」和「建单消息最终可消费」绑在一起。
- `B`：入口只做限流并写排队状态，投递普通消息后立刻返回订单号，库存扣减与一人一单移到消费者里执行。

应用内第二层限流是 Redis 滑动窗口，`seckill.rate-limit.sliding-window` 控制窗口长度和同一用户的请求上限。

## 5. 可选：OpenResty 网关

`src/main/resources/openresty/` 是网关示例，不起也能跑，只是少了两层：秒杀入口的令牌桶（1000/s，桶容量 3000）和商铺详情的共享内存 L1 缓存（TTL 600s ± 60s）。

```bash
cp -r src/main/resources/openresty/lua /usr/local/openresty/nginx/lua
```

把 `nginx.conf` 里的 `lua_package_path` 和 `upstream` 改成实际路径与端口（默认 8081），reload 后用 80 端口访问。

## 注意事项

- 消费失败超过 `rocketmq.consumer.max-reconsume-times`（默认 16）会进 `%DLQ%<consumer-group>` 死信队列，日志里记了消息 ID 和重试次数。
- 定时任务 `SeckillReconcileTask` 负责对账：关超时未支付单、补缺失的订单关系、按有效订单校正 Redis 与 MySQL 库存。
- 想彻底重置环境用 `docker compose down -v`，日常停止不要带 `-v`。

接口示例：`curl -i http://127.0.0.1:8081/shop-type/list`
