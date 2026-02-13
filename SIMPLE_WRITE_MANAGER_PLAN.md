# Write Manager 轻量方案（包级能力）

> 目标：不是做独立服务，而是做一个可嵌入业务项目的 Go 包，统一写操作流程并提供公共能力。

## 1. 设计边界

- **做什么**
  - 提供统一的 `Writer` 抽象。
  - 提供 `WriteManager` 编排入口。
  - 提供公共能力：参数校验、重试、超时、日志、指标、错误归一化、按 type 路由。
- **不做什么**
  - 不做网关。
  - 不做鉴权。
  - 不做独立部署服务。

## 2. 核心接口

```go
type WriteRequest struct {
    Target   string            // 目标资源，如 mysql.main / redis.cache
    Type     string            // 业务类型，如 user_profile_update
    Payload  any               // 业务数据
    Meta     map[string]string // trace_id / source / operator 等可选上下文
}

type WriteResult struct {
    Success bool
    Message string
    Data    any
}

type Writer interface {
    Name() string
    Supports(target string) bool
    Write(ctx context.Context, req WriteRequest) (WriteResult, error)
}

type WriteManager interface {
    Register(writer Writer)
    Write(ctx context.Context, req WriteRequest) (WriteResult, error)
}
```

## 3. 分层思路

- `manager`：统一入口、路由、通用流程。
- `writer`：不同存储实现（`mysqlWriter`、`redisWriter`）。
- `handler`：同一个 writer 下按 `Type` 做业务逻辑分发。

示例：
- `mysqlWriter` 内部维护 `map[type]MysqlTypeHandler`
- `redisWriter` 内部维护 `map[type]RedisTypeHandler`

这样可以实现：
1. 先按 target 找 writer（mysql / redis）。
2. 再按 type 走具体处理逻辑（不同库、不同表、不同 key 规则）。

## 4. 推荐执行流程

`WriteManager.Write()` 建议固定以下步骤：

1. 基础校验（target/type/payload 非空、格式合法）。
2. 查找可处理该 target 的 writer。
3. 执行通用中间能力：
   - 超时控制（`context.WithTimeout`）
   - 可选重试（仅对可重试错误）
   - 统一日志（开始/结束/耗时/失败原因）
   - 统一指标（成功率、延迟、按 writer/type 维度）
4. 调用 writer.Write。
5. 统一错误包装并返回。

## 5. type 路由建议

在每种 writer 下使用“注册式路由”，避免 if-else 膨胀：

```go
type MysqlTypeHandler interface {
    Type() string
    Handle(ctx context.Context, req WriteRequest, db *sql.DB) (WriteResult, error)
}
```

- `mysqlWriter.RegisterHandler(h MysqlTypeHandler)`
- `redisWriter.RegisterHandler(h RedisTypeHandler)`

这样新增类型只要加 handler，不用改 manager 主流程。

## 6. 错误模型（建议）

定义统一错误码，便于调用方处理：

- `ErrInvalidRequest`：参数非法
- `ErrWriterNotFound`：找不到对应 writer
- `ErrTypeNotSupported`：writer 不支持该 type
- `ErrTimeout`：超时
- `ErrRetryExhausted`：重试耗尽
- `ErrBackend`：底层存储错误

并提供 `Wrap(code, msg, cause)`，保证日志与返回一致。

## 7. 最小落地目录（包模式）

```text
write_manager/
  manager/
    manager.go
    options.go
  writer/
    writer.go
    mysql_writer.go
    redis_writer.go
  errors/
    errors.go
  middleware/
    retry.go
    timeout.go
    logging.go
    metrics.go
  examples/
    basic/main.go
```

## 8. 先做 MVP（1~2 天）

建议先实现最小闭环：

1. `Writer` / `WriteManager` 接口。
2. `mysqlWriter`、`redisWriter` 空实现 + type 路由。
3. manager 的超时 + 日志 + 基础错误归一化。
4. 一个示例 type：
   - `mysql`: `user_profile_update`
   - `redis`: `cache_set`

完成后再逐步加重试、指标、批量写、事务等增强能力。

## 9. 你这个场景下的关键收益

- 各业务写逻辑接入方式一致，降低心智负担。
- 通用能力（日志/错误/重试）一次实现，全局复用。
- 新增存储类型或写 type 只加实现，不改主链路。
- 可以先在个人笔记本完成设计与 demo，再迁回内网项目。
