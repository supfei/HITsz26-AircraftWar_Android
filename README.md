# AircraftWarDemo

《软件构造实践》课程项目 —— 将《软件构造》课程中的 **Java Swing 桌面版飞机大战** 迁移至 **Android 移动端**，在保留核心玩法与面向对象设计的前提下，完成界面重构、触控与音频适配、本地/云端排行榜及双人对战等扩展功能。

---

## 功能概览

- **单机模式**：简单 / 普通 / 困难三种难度，触控拖动英雄机，道具、Boss 战、背景滚动
- **本地排行榜**：Room（SQLite）持久化，支持查看与删除历史成绩
- **全球排行榜**：游戏结束后上传分数，从服务端拉取 Top 100
- **双人对战**：创建/加入 6 位房间号，轮询同步对手分数与血量，对战结束后展示胜负结果
- **音频**：背景音乐（普通 / Boss）、射击/爆炸等短音效，支持开关并与生命周期联动

> **说明**：仓库包含 Android 客户端源码、`.env.example` 环境模板及 `server/app_exmple.js` 服务端配置模板。课程演示用的云服务器并非 7×24 小时常驻，**克隆后需自行部署后端** 才能使用排行榜与联机功能；单机模式无需联网即可运行。

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 客户端 | Java 11、Android SDK 30+、SurfaceView、AppCompatActivity、Room、Retrofit + OkHttp + Gson |
| 服务端 | Node.js、Express、mysql2、cors |
| 数据库 | MySQL（`scores`、`match_rooms` 等表，启动时自动建表） |

### 设计模式与架构要点

- **平台无关逻辑复用**：`AbstractGame` / `GameController` 体系、敌机工厂、碰撞检测、英雄机单例等自 Swing 版迁移
- **策略模式**：多种射击策略（直线、散射、环射等）
- **观察者模式**：游戏状态通知
- **对象池**：子弹对象池，缓解高频 GC 导致的卡顿
- **绘制与输入分离**：`GameView`（SurfaceView + 固定时间步循环）+ `TouchController`

---

## 仓库结构

```
AircraftWarDemo/
├── app/                          # Android 客户端
│   └── src/main/java/com/example/aircraftwardemo/
│       ├── activity/             # 主菜单、难度选择、游戏、结算、排行榜、联机大厅等
│       ├── controller/           # Easy / Normal / Hard 游戏控制器
│       ├── view/                 # GameView（Surface 绘制与游戏线程）
│       ├── model/                # 飞机、子弹、道具等实体
│       ├── network/              # 分数上传、全球榜、联机会话管理
│       ├── dao/                  # Room 本地成绩存储
│       ├── manager/              # 图片、音频等资源管理
│       ├── pool/                 # 对象池
│       └── strategy/             # 射击策略
├── server/
│   └── app_exmple.js             # 服务端示例（Express + MySQL，默认 8080 端口）
├── .env.example                  # 客户端 API 地址配置模板
└── gradle/                       # Gradle 与依赖版本
```

---

## 快速开始

### 环境要求

- **Android 客户端**：Android Studio（推荐）、JDK 11、Android SDK（`minSdk 30`，`targetSdk 36`）
- **服务端**（可选）：Node.js、MySQL 5.7+ / 8.x

### 1. 配置客户端

```bash
cp .env.example .env
```

编辑 `.env`，填写可访问的服务器地址（**`SCORE_API_BASE_URL` 必须以 `/` 结尾**）：

```env
PUBLIC_IP=192.168.1.100
SCORE_API_PORT=8080
# 或直接指定完整地址：
# SCORE_API_BASE_URL=http://192.168.1.100:8080/
```

构建时 Gradle 会将上述配置注入 `BuildConfig.SCORE_API_BASE_URL`，供 Retrofit 使用。

用 Android Studio 打开项目根目录，连接真机或模拟器后运行 `app` 模块即可。真机调试联机/排行榜时，请确保手机与服务器网络互通。

### 2. 部署服务端

进入 `server/` 目录，修改 `app_exmple.js` 顶部的 `DB_CONFIG`：

```javascript
const DB_CONFIG = {
  host: "127.0.0.1",
  port: 3306,
  user: "your_user",
  password: "your_password",
  database: "aircraft_war",
  charset: "utf8mb4",
};
```

安装依赖并启动（仓库未附带 `package.json` 时可手动初始化）：

```bash
cd server
npm init -y
npm install express mysql2 cors
node app_exmple.js
```

健康检查：浏览器或 `curl` 访问 `http://<服务器IP>:8080/`，应返回 JSON：

```json
{ "ok": true, "message": "AircraftWar server is running." }
```

**部署注意**：

- 在云服务器安全组 / 防火墙中放行 TCP **8080**（或你实际使用的端口）
- MySQL 需提前创建数据库，脚本启动时会自动建表
- 调试环境使用 HTTP 时，客户端已在 `AndroidManifest.xml` 中设置 `android:usesCleartextTraffic="true"`；生产环境建议改用 HTTPS

---

## API 接口

与客户端 Retrofit 路径一致，基址示例：`http://<host>:8080/`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/` | 健康检查 |
| POST | `/api/score/submit` | 提交分数 |
| GET | `/api/score/top100` | 全球排行榜 Top 100 |
| GET | `/api/score/search` | 按条件查询分数 |
| POST | `/api/match/room/create` | 创建对战房间 |
| POST | `/api/match/room/join` | 加入房间 |
| GET | `/api/match/room/status` | 轮询房间状态 |
| POST | `/api/match/room/start` | 房主开始游戏 |
| POST | `/api/match/score/sync` | 对局中同步分数/血量 |
| POST | `/api/match/score/finish` | 结束对战 |
| GET | `/api/match/result/status` | 获取对战结果 |
| POST | `/api/match/room/cleanup` | 清理房间 |

联机流程简述：大厅创建/加入房间 → 约 1.2s 轮询房间状态 → 房主开始后双方进入 `MainActivity` → 对局内约 400ms 同步分数 → 本地结束后调用 `finish` → `MatchResultActivity` 展示结果。

---

## 主要界面

| Activity | 功能 |
|----------|------|
| `MainMenuActivity` | 启动页：单机、联机、排行榜入口 |
| `DifficultySelectActivity` | 难度选择 |
| `MainActivity` | 游戏主界面（全屏 SurfaceView） |
| `GameOverActivity` | 结算、输入昵称、上传成绩 |
| `GlobalRankingActivity` | 本地榜 + 全球榜（RecyclerView） |
| `OnlineLobbyActivity` | 联机大厅：创建/加入房间 |
| `MatchResultActivity` | 对战胜负结果 |

---

## 常见问题

| 现象 | 处理 |
|------|------|
| 网络请求失败 | 检查 `.env` 中 API 地址、服务器端口与安全组；确认 `baseUrl` 以 `/` 结尾 |
| 404 或路径重复 `/api/api/...` | 统一 Retrofit `baseUrl` 与 `@GET`/`@POST` 相对路径写法 |
| 联机一方不进入游戏 | 确认访客已保存 `room_id`，轮询 `started` 字段；检查网络是否中断 |
| JSON 字段映射为 null | 对齐客户端 DTO 与服务端 JSON 字段名，或使用 `@SerializedName` |

---

## 已知限制与后续改进

- 对战同步采用 **HTTP 短轮询**，实时性与服务器负载有权衡空间，可改为 WebSocket / SSE
- 服务端 **无鉴权**，演示环境可用；生产需增加 Token、频控或签名校验
- 6 位数字房间号已通过数据库查重，理论上仍有冲突可能

---

## 许可证

本项目为课程实践作业，仅供学习交流使用。排行榜与联机功能需自行部署后端后使用。
