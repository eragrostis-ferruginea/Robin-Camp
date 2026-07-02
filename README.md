# Movie Rating API

一个用 Go 实现的电影评分服务，提供电影的创建、检索、列表分页以及评分上报与聚合能力。创建电影时会同步调用第三方票房（Box Office）Mock API 进行数据整合。项目以 `openapi.yml` 为接口合同，通过 Docker Compose 一键拉起应用与数据库。

---

## 目录

- [功能特性](#功能特性)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [快速开始](#快速开始)
- [环境变量](#环境变量)
- [API 接口](#api-接口)
- [数据库设计与选型](#数据库设计与选型)
- [后端服务设计与选型](#后端服务设计与选型)
- [Make 命令](#make-命令)
- [端到端测试](#端到端测试)
- [可优化与改进方向](#可优化与改进方向)

---

## 功能特性

- **电影管理**
  - `POST /movies`：创建电影，成功后同步调用票房接口合并数据；上游失败时不阻塞创建，`boxOffice` 置空。
  - `GET /movies`：支持 `q / year / genre / distributor / budget / mpaRating / limit / cursor` 组合查询与游标分页。
- **评分系统**
  - `POST /movies/{title}/ratings`：基于 `(movieTitle, raterId)` 的 Upsert 语义，重复提交覆盖旧值。
  - `GET /movies/{title}/rating`：返回 `{average, count}` 聚合结果。
- **健康检查**：`GET /healthz` 返回 200，供容器探活。
- **鉴权**：写操作通过 `Authorization: Bearer <token>` 校验静态 Token。
- **配置全部来自环境变量**，无硬编码密钥或 URL。

## 技术栈

| 层级 | 选型 | 说明 |
| --- | --- | --- |
| 语言 | Go 1.21 | 编译型、并发友好、部署镜像小 |
| Web 框架 | Gin | 路由、中间件、绑定等开箱即用 |
| 数据库 | PostgreSQL 15 | 关系型主存储，支持 JSONB 与 CHECK 约束 |
| 迁移工具 | golang-migrate | 版本化 SQL 迁移，启动时自动执行 |
| 配置 | godotenv + `os.Getenv` | 从 `.env` 与环境变量注入 |
| 容器 | Docker + Docker Compose | 多阶段构建，Compose 编排应用与数据库 |

## 项目结构

```
.
├── cmd/api/main.go              # 入口：装配依赖、注册路由、启动 HTTP 服务
├── internal/
│   ├── config/                  # 配置加载（环境变量）
│   ├── handlers/                # HTTP 处理器（movie / health）
│   ├── middleware/              # Bearer Token 鉴权中间件
│   ├── models/                  # 领域模型（Movie / Rating / BoxOffice）
│   ├── repository/              # 数据访问层 + 迁移执行
│   ├── service/                 # 业务逻辑（movie / rating / boxoffice）
│   └── migrations/              # SQL 迁移文件（up/down）
├── openapi.yml                  # 主服务 API 合同
├── boxoffice.openapi.yml        # 第三方票房接口合同
├── mock-boxoffice.json          # 票房 Mock 样本数据
├── Dockerfile                   # 多阶段构建
├── docker-compose.yml           # 应用 + 数据库编排
├── Makefile                     # 构建 / 运行 / Docker 操作
├── e2e-test.sh                  # 端到端测试脚本（curl + jq）
└── .env.example                 # 环境变量示例
```

整体采用**清晰分层**：`handlers → service → repository → DB`，每层都通过接口定义依赖，便于测试与替换。

---

## 快速开始

### 前置依赖

- Docker、Docker Compose
- bash、curl、jq（运行 E2E 测试）

### 一键启动

```bash
# 1. 准备环境变量
cp .env.example .env
# 编辑 .env，填入 AUTH_TOKEN / BOXOFFICE_URL / BOXOFFICE_API_KEY

# 2. 启动服务（构建镜像 + 拉起 db 与 api，等待健康检查）
make docker-up
# 或：docker compose up -d --build

# 3. 验证服务
curl http://127.0.0.1:8080/healthz
```

### 端到端测试

```bash
make test-e2e
# 或：./e2e-test.sh
```

### 清理

```bash
make docker-down
```

---

## 环境变量

| 变量 | 说明 | 默认值 |
| --- | --- | --- |
| `PORT` | 服务监听端口 | `8080` |
| `AUTH_TOKEN` | 写操作所需的静态 Bearer Token | 无（必填） |
| `DB_URL` | PostgreSQL 连接字符串 | `postgres://postgres:postgres@localhost:5432/movies?sslmode=disable` |
| `BOXOFFICE_URL` | 第三方票房 Mock 基础 URL | 无 |
| `BOXOFFICE_API_KEY` | 调用票房接口的静态 Key | 无 |

> 所有配置均通过环境变量注入，禁止在代码中硬编码。

---

## API 接口

完整合同见 [openapi.yml](openapi.yml)，摘要如下：

| 方法 | 路径 | 鉴权 | 说明 |
| --- | --- | --- | --- |
| GET | `/healthz` | 否 | 健康检查 |
| POST | `/movies` | Bearer Token | 创建电影，同步合并票房数据，返回 201 + `Location` |
| GET | `/movies` | Bearer Token | 列表与搜索，支持多过滤条件与游标分页 |
| POST | `/movies/{title}/ratings` | `X-Rater-Id` | 提交评分（Upsert） |
| GET | `/movies/{title}/ratings` | 否 | 获取评分聚合 `{average, count}` |

### 创建电影示例

```bash
curl -X POST http://127.0.0.1:8080/movies \
  -H "Authorization: Bearer $AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Inception",
    "releaseDate": "2010-07-16",
    "genre": "Sci-Fi",
    "distributor": "Warner Bros. Pictures",
    "budget": 160000000,
    "mpaRating": "PG-13"
  }'
```

### 提交评分示例

```bash
curl -X POST "http://127.0.0.1:8080/movies/Inception/ratings" \
  -H "X-Rater-Id: user_456" \
  -H "Content-Type: application/json" \
  -d '{"rating": 4.5}'
```

---

## 数据库设计与选型

### 选型理由

选择 **PostgreSQL** 作为主存储，主要考虑：

1. **数据一致性强**：电影与评分之间存在外键关系（评分引用电影标题），需要数据库层面保证引用完整性，关系型数据库天然适合。
2. **JSONB 支持**：票房数据结构较灵活（`revenue` 内含可变字段），用 `JSONB` 列存储可避免频繁加列，同时支持后续按 JSON 字段查询索引。
3. **CHECK 约束**：评分值限定在 `{0.5, 1.0, …, 5.0}` 集合内，直接在表结构上用 `CHECK` 约束兜底，避免应用层校验遗漏导致脏数据。
4. **生态成熟**：`golang-migrate` + `lib/pq` 在 Go 社区使用广泛，迁移与连接管理稳定可靠。

### 表结构设计

#### `movies` 表

| 列 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `id` | VARCHAR(255) | PRIMARY KEY | 应用生成的电影 ID |
| `title` | VARCHAR(255) | NOT NULL, UNIQUE | 电影标题，作为业务主键 |
| `release_date` | DATE | NOT NULL | 北美上映日期 |
| `genre` | VARCHAR(100) | NOT NULL | 类型 |
| `distributor` | VARCHAR(255) | nullable | 发行商 |
| `budget` | BIGINT | nullable | 制作预算（USD） |
| `mpa_rating` | VARCHAR(10) | nullable | MPA 分级 |
| `box_office` | JSONB | nullable | 票房聚合信息 |
| `created_at` / `updated_at` | TIMESTAMP | 默认当前时间 | 审计字段 |

索引：`title`、`release_date` 年份表达式、`genre`、`distributor`，覆盖常见查询路径。

#### `ratings` 表

| 列 | 类型 | 约束 | 说明 |
| --- | --- | --- | --- |
| `movie_title` | VARCHAR(255) | NOT NULL, FK → movies(title) ON DELETE CASCADE | 关联电影 |
| `rater_id` | VARCHAR(255) | NOT NULL | 评分者标识 |
| `rating` | FLOAT | NOT NULL, CHECK ∈ {0.5 … 5.0} | 评分值 |
| `created_at` / `updated_at` | TIMESTAMP | 默认当前时间 | 审计字段 |

主键为 `(movie_title, rater_id)` 复合主键，天然支撑 Upsert 语义（`ON CONFLICT DO UPDATE`）。聚合查询通过 `AVG(rating)` + `COUNT(*)` 完成。

### 迁移策略

- 使用 `golang-migrate` 的版本化 SQL 文件（`000001_*.up.sql` / `*.down.sql`）。
- 应用启动时自动执行 `migrate.Up()`，无需手动初始化。
- 对脏状态（dirty）有兜底处理：检测到 dirty 时调用 `Force` 强制版本，避免单次失败导致服务无法启动。

---

## 后端服务设计与选型

### 框架与语言选型

- **Go + Gin**：Go 编译产物为单一二进制，容器镜像小、启动快、内存占用低，适合容器化部署；Gin 提供路由分组、中间件链、JSON 绑定等基础能力，开发效率与性能均衡。
- **标准库 `database/sql` + `lib/pq`**：不引入重型 ORM，SQL 手写更可控，便于按 `openapi.yml` 精确控制字段与分页行为。

### 分层架构

```
handlers (HTTP 适配层)
   ↓
service (业务逻辑层，编排 repository 与外部调用)
   ↓
repository (数据访问层，封装 SQL)
   ↓
PostgreSQL
```

- 每层通过**接口**定义依赖（如 `MovieService`、`MovieRepository`、`BoxOfficeService`），实现可替换、可测试。
- `BoxOfficeService` 在配置缺失时返回 `nil` 实例，`MovieService` 通过 nil 检查实现降级，保证主流程不被外部依赖拖垮。

### 关键设计点

1. **票房集成降级**：`BoxOfficeService.GetBoxOfficeData` 在网络错误、非 200 响应、JSON 解析失败等情况下均返回 `nil`，`CreateMovie` 仅在拿到非 nil 数据时填充 `boxOffice`，**上游任何异常都不阻塞电影创建**。
2. **Upsert 语义**：评分表复合主键 + `ON CONFLICT (movie_title, rater_id) DO UPDATE`，一条 SQL 完成插入或覆盖，避免应用层先查后写带来的竞态。
3. **鉴权中间件**：通过 Gin 中间件统一校验 `Authorization: Bearer <token>`，写路由组统一挂载，避免在每个 handler 重复校验。
4. **游标分页**：`List` 查询多取一行（`LIMIT n+1`）判断是否有下一页，避免额外 `COUNT` 查询。
5. **配置合规**：所有端口、Token、URL 均从环境变量读取，`.env.example` 列出全部所需变量。

---

## Make 命令

| 命令 | 说明 |
| --- | --- |
| `make build` | 编译生成 `bin/movie-rating-api` |
| `make run` | 编译并运行 |
| `make test` | 运行 Go 单元测试 |
| `make fmt` | 格式化代码 |
| `make deps` | 下载 / 整理依赖 |
| `make docker-up` | 构建并启动全部容器 |
| `make docker-down` | 停止并清理容器 |
| `make docker-restart` | 重启容器 |
| `make docker-logs` | 查看全部日志 |
| `make docker-logs-api` | 查看 API 日志 |
| `make docker-logs-db` | 查看数据库日志 |
| `make docker-status` | 查看容器状态 |

---

## 端到端测试

`e2e-test.sh` 使用 `curl` + `jq` 编写，分 6 个阶段：

1. **环境与健康检查**：校验 `AUTH_TOKEN` 等环境变量，访问 `/healthz`。
2. **基础 CRUD**：创建电影（含票房命中与未命中两种场景）、列表查询。
3. **评分系统**：新评分（201）、覆盖评分（200 Upsert）、聚合计算校验。
4. **搜索与分页**：`q`、`year`、`genre` 过滤与 `limit` + `cursor` 分页。
5. **鉴权与权限**：缺失/无效 Token 与 `X-Rater-Id` 应返回 401。
6. **错误处理**：不存在电影返回 404、非法评分值返回 422、非法 JSON 与日期格式校验。

---

## 可优化与改进方向

在完成主干功能后，复盘代码可以发现以下可继续打磨的点：

### 1. 配置注入的一致性

`cmd/api/main.go` 中数据库连接字符串与监听端口存在硬编码（直接写死了 `localhost:5432` 与 `:9090`），并未真正使用 `config.Config` 中的 `DBURL` 与 `Port`。这会导致 Docker Compose 环境下连不上 `db` 主机、端口与合同不一致。应统一改为：

```go
db, err := repository.InitDB(cfg.DBURL)
serverAddr := ":" + cfg.Port
```

### 2. 读操作公开化

`openapi.yml` 中 `GET /movies` 未声明 `security`，按合同应为公开读；当前实现将其放进了鉴权路由组。应将读路由拆出，仅写操作要求 Bearer Token。

### 3. 评分聚合的四舍五入

合同要求 `average` 四舍五入保留 1 位小数。当前 `GetAggregateByMovie` 直接返回 `AVG(rating)` 原始值，未做舍入。应在 service 层用 `math.Round(avg*10)/10` 处理，或在 SQL 中用 `ROUND(AVG(rating)::numeric, 1)`。

### 4. 游标分页的真实化

`MovieRepository.List` 中 `nextCursor` 被硬编码为字符串 `"next"`，无法用于翻页。应基于最后一行的排序字段（如 `(release_date, title)` 或 `id`）生成 Base64 编码的游标，并在下一页查询时解析为 `WHERE` 条件。

### 5. 错误响应模型对齐合同

`openapi.yml` 定义错误体为 `{code, message, details}`，而代码中大量使用 `gin.H{"error": "..."}`。应统一封装错误响应函数，输出 `{code: "BAD_REQUEST", message: "..."}`，并补齐 422 等状态码（当前非法评分返回 400）。

### 6. 路由命名一致性

合同中聚合评分接口为 `GET /movies/{title}/rating`（单数），代码注册的是 `/movies/{title}/ratings`（复数）。应与合同对齐，避免端到端测试与文档不一致。

### 7. 票房接口鉴权方式

`boxoffice.openapi.yml` 要求通过请求头 `X-API-Key` 鉴权，当前实现把 key 拼到了 query string（`apikey=...`）。应改为请求头传递，符合第三方合同。

### 8. 票房集成的健壮性

可补充：
- **超时与重试**：当前仅 10s 超时，无重试。可加入有限次指数退避重试（仅对 5xx / 网络错误）。
- **熔断**：上游持续故障时短路，避免每次创建电影都等待超时。
- **异步化**：将票房合并改为创建后异步任务（消息队列或后台 worker），进一步降低写延迟。

### 9. Dockerfile 安全与体积

合同要求最终镜像**非 root 运行**，当前 `Dockerfile` 未创建非 root 用户。应增加：

```dockerfile
RUN adduser -D -u 10001 appuser
USER appuser
```

并可考虑使用 `scratch` 或 `gcr.io/distroless/static` 作为运行时基础镜像进一步缩减体积。

### 10. 可观测性

- **结构化日志**：当前用 `log.Printf`，可引入 `slog` 或 `zerolog` 输出 JSON 结构化日志，带 `request_id`、`movie_title` 等字段。
- **指标**：暴露 Prometheus 指标（QPS、p99 延迟、票房调用成功率/延迟、DB 连接池占用）。
- **链路追踪**：接入 OpenTelemetry，串起 HTTP → service → repository → boxoffice 的调用链。

### 11. 测试覆盖

- 当前仅有 E2E 脚本，缺少 Go 单元测试。应为 `service` 与 `repository` 层补充表驱动测试，使用 `sqlmock` 或测试容器（testcontainers）隔离 DB。
- `Makefile` 缺少 `test-e2e` 目标（合同要求），应补齐：`test-e2e: ; ./e2e-test.sh`。

### 12. 数据模型的小改进

- `movies.id` 当前由应用用标题前缀 + 毫秒时间戳生成，存在极低概率碰撞且可预测。可改用 `UUID v7`（时间有序，利于 B-tree 局部写入）或数据库 `gen_random_uuid()`。
- `ratings.movie_title` 作为外键引用 `movies(title)`，若后续允许修改电影标题需级联更新；可考虑改用 `movie_id` 作为外键，标题仅作业务展示字段。
