# SQLite 和 MySQL 怎么选型？

### 面试回答

SQLite 和 MySQL 的核心区别，不是 SQL 语法，而是**部署模型和状态共享方式**。

- **SQLite** 是嵌入式本地数据库，通常就是一个本地 `.db` 文件，应用直接读写，适合单机、本地工具、桌面应用、本地 Agent 状态索引等场景。
- **MySQL** 是独立数据库服务，适合线上后端、多实例部署、多个服务节点共享同一份状态。

如果是单机、本地状态，优先考虑 SQLite；如果状态需要被多个实例共享，优先考虑 MySQL / PostgreSQL。

### 核心对比

| 维度 | SQLite | MySQL |
|---|---|---|
| 部署方式 | 嵌入式，本地文件 | 独立数据库服务 |
| 是否需要单独启动服务 | 不需要 | 需要 |
| 数据访问 | 应用直接读写本地文件 | 通过网络 / Socket 访问 |
| 单机本地工具 | 很适合 | 偏重 |
| 多实例共享状态 | 不适合 | 很适合 |
| 写并发能力 | 较弱 | 更强 |
| 用户权限 / 运维能力 | 简单 | 更完整 |
| 典型场景 | 桌面 App、本地工具、本地 Agent | Web 后端、线上 Agent、分布式系统 |

### 我的追问 ⭐

#### 单个 `userId + sessionId` 内明明是串行的，为什么线上 Agent 还更适合 MySQL？

关键不在“同一个 Session 是否并发”，而在**线上服务通常是多实例部署，状态需要跨实例共享**。

例如：

```text
用户请求
   ↓
负载均衡
   ↓
Agent 实例 1 / 2 / 3
```

第一次请求可能落到实例 1，下一次请求可能落到实例 2。

如果使用 SQLite：

```text
实例1 → /local/state.db
实例2 → /local/state.db
```

每个实例本地文件彼此独立，实例 2 不一定能读到实例 1 的 Session 状态。

而 MySQL 是共享服务：

```text
Agent实例1 ─┐
Agent实例2 ─┼→ MySQL
Agent实例3 ─┘
```

所以即使一个 Session 内完全串行，只要服务是多实例部署，MySQL 仍然更适合作为共享状态库。

### 最终心智模型

```text
SQLite
= 状态跟着某一台机器走

MySQL
= 状态放在独立服务里，多个实例都能访问
```

因此选型时不要只问“有没有并发”，还要问：

> **这份状态是不是只属于当前机器，还是需要被多个服务实例共享？**

### 结合 Agent 场景

对于线上 Agent，可以这样分：

```text
MySQL
├── agent_session
│   └── user_id / session_id / status / updated_at
└── agent_event
    └── session_id / turn_id / seq / event_type / payload

OSS
└── CSV / 图片 / PDF 等大文件

Redis（可选）
└── 临时运行状态、缓存、锁
```

如果系统只是单机运行、状态完全本地化，SQLite 也可以；如果 Agent 多实例部署，需要共享 Session / Event / Checkpoint，则优先 MySQL。
