# Telegraph 图床

基于 Cloudflare Worker 和 Telegram Bot API 的图床 / 视频床 / 文件床服务

> 本仓库 fork 自 [0-RTT/telegraph](https://github.com/0-RTT/telegraph)（上游基线：2026-09-13），
> 在此之上做了独立维护与增强。所有改动均向后兼容，可无缝替换上游 `_worker.js`。
> 当前版本：**v2.2.0**（页面底部与管理页头部会显示版本号徽章）

---

## 本 Fork 相比上游的改动

### 1. 数据与域名彻底解耦（换域名不再 404）

上游把完整 URL（`https://example.com/1734567890123.jpg`）存进 D1 且作为主键，
一旦更换域名，查询字符串对不上，历史文件**全部 404**。本 Fork 的修复分两层：

- **存储层**：新上传只存「裸 key」`1734567890123-a1b2c3d4.jpg`，域名不再入库。
- **读取层**：查询时一次 `IN` 同时匹配三种形态 —— 裸 key / 当前域名完整 URL / 请求域名完整 URL。
  因此**历史老数据无需任何改动就能正常访问**，换域名、用 `*.workers.dev` 访问都不会 404。
- 缓存键统一为规范化 URL，多个入口共用一份缓存。
- 删除接口同样做归一化，完整 URL 与裸 key 都能正确删除。

> 新旧两种形态**永久共存**，不存在「必须迁移」这回事。
> 想统一数据形态的话见下方「数据形态统一（可选）」，在 D1 控制台跑几条 SQL 即可，项目里不内置迁移功能。

### 2. 上传格式白名单大幅扩充（47 → 61 项）

按分类整理并加了注释，涵盖图片 / 视频 / 音频 / 压缩包 / 文档：

| 分类 | 格式 |
|------|------|
| 图片 | jpg jpeg png gif webp bmp tif tiff ico avif heic heif svg |
| 视频 | mp4 m4v webm ogv avi mov mkv flv wmv mpg mpeg 3gp ts |
| 音频 | mp3 m4a aac wav flac ogg oga opus wma aiff |
| 压缩包 | zip rar 7z tar gz tgz bz2 xz |
| 文档 | pdf txt md csv doc docx xls xlsx ppt pptx epub rtf |

- `ALLOWED_EXTENSIONS` 由同一张表派生，加一处即同时生效于上传校验与下载 `Content-Type`
- `svg` 仍强制 `Content-Disposition: attachment` + 严格 CSP，杜绝存储型 XSS
- 严禁加入 `html / htm / js / mjs / css / xml / json / wasm / php`（源码注释中已标注）
- 修复管理页 `<source type="video/mkv">` 这类非法 MIME（改为查映射表取真实 MIME）

### 3. 管理页游标分页（修复深翻页性能问题）

上游用 `ORDER BY url DESC LIMIT 50 OFFSET n`，翻到第 N 页要**读取并丢弃**前 `50×N` 行，
百万级库深翻页既慢又疯狂消耗 D1 rows read。本 Fork 改为游标分页：

```sql
-- 下一页：索引范围扫描，直接定位锚点
SELECT url FROM media WHERE url  < ? ORDER BY url DESC LIMIT 51
-- 上一页：反向取回后在内存反转，同样走索引
SELECT url FROM media WHERE url  > ? ORDER BY url ASC  LIMIT 51
```

- 每页恒定只读 51 行，**翻到多深都一样快**；实测 30 万行库深翻页 0.03 ms vs OFFSET 3.46 ms（约 100 倍）
- 已校验：与 OFFSET 结果逐行一致，前后翻页无重复、无遗漏
- 双向导航 + 「回到最新」按钮；每页多取 1 行判断是否有下一页，省掉额外 COUNT 查询

### 4. 管理页总数缓存

原实现每翻一页就 `SELECT COUNT(*)`，百万行表等于每次扫描近百万行、消耗近百万 rows read。
改为 isolate 级 60 秒缓存，误差可接受，isolate 回收后自动重建。

### 5. 版本号显示

`APP_VERSION` 常量显示在首页底部与管理页 header 的徽章上，便于确认线上跑的是哪一份代码。

### 6. 小修

- `ADMIN_PATH` 增加默认值 `admin`（上游未配置时路径会变成 `/undefined`）

---

## 功能特点（继承自上游）

### 核心功能
- 🔐 可选的访客验证功能（Basic Auth）
- 🗜️ 可选的图片压缩功能（默认开启，支持前端切换）
- 📦 可选的文件大小限制（默认 20MB，可通过环境变量配置）
- 📁 支持图片、视频等常见媒体格式上传（白名单校验，防止存储型 XSS）
- 📤 支持多文件上传、拖拽上传和粘贴上传（Ctrl+V）
- 🔄 哈希校验避免重复上传
- 🚀 多文件并发上传（默认 4 路并发）

### 管理功能
- 📋 支持查看本地历史记录
- 🖼️ 图库管理界面，支持批量操作
- 🗑️ 支持批量删除文件（同步删除数据库记录和 CDN 缓存）
- ⏰ 显示文件上传时间
- 📋 支持多种格式复制链接（URL、BBCode、Markdown）
- 🧭 游标分页，百万级数据翻页不卡顿

### 界面与交互
- 🪟 **苹果液态玻璃（Liquid Glass）视觉风格**
  - 卡片、按钮、下拉菜单、分页器、输入框全部使用 `backdrop-filter` 毛玻璃
  - 多层 `inset box-shadow` 模拟玻璃边缘高光与折射
  - 卡片支持鼠标跟随镜面高光（`pointermove` 实时更新 `--mx / --my`）
  - 卡片入场动画：`cubic-bezier(0.22, 1, 0.36, 1)` 缓动
- 🔔 **右上角玻璃 Toast 通知**
  - 淡入淡出（opacity + translateY + scale），约 320–420ms 缓动
  - 提示文案精简（如"上传成功"/"上传失败"/"复制成功"/"命中缓存"）
  - 支持 `success` / `error` / `warning` / `info` 四种状态，用彩色小圆点区分
  - 多条通知在右上角自动堆叠
- 🪟 **玻璃确认框**：删除等危险操作使用玻璃风格弹窗替代原生 `confirm()`
- 📱 响应式设计，移动端 Toast 自动撑满左右两侧
- 🏷️ 版本号徽章

### 性能优化
- ⚡ Cloudflare Cache API 缓存支持
- 🎨 懒加载和骨架屏优化
- 🌅 Bing 每日壁纸背景（自动轮播）
- 🔁 获取文件路径自动重试（最多 3 次，带退避）
- 🪶 首页与管理页均零外部依赖（无 jQuery / Bootstrap / FontAwesome / toastr / 任何 CDN 资源）

### 存储方式
- 📡 基于 Telegram Bot API 的文件存储
- 💾 使用 Cloudflare D1 数据库存储文件映射关系
- 🎯 通过 fileId 实现文件访问
- 🔗 url 以裸 key 形式存储，与域名解耦，新旧形态永久共存

---

## 部署

> ⚠️ 虽然项目使用了 Worker 的缓存 API，但仍建议配置好 **边缘 TTL** 并开启 **访客验证**，防止被刷导致扣费。

### 环境变量

在 Cloudflare Workers 中配置以下变量：

| 变量名 | 说明 | 必填 | 默认值 | 示例 |
|--------|------|:--:|:--:|------|
| DOMAIN | 自定义域名 | 是 | - | example.workers.dev |
| DATABASE | D1 数据库绑定变量名称 | 是 | - | DATABASE |
| TG_BOT_TOKEN | Telegram Bot Token | 是 | - | 123456789:ABCdefGHIjklMNOpqrsTUVwxyz |
| TG_CHAT_ID | Telegram 频道/群组 ID | 是 | - | -100xxxxxxxxxx |
| USERNAME | 管理员用户名 | 是 | - | admin |
| PASSWORD | 管理员密码 | 是 | - | password123 |
| ADMIN_PATH | 管理后台路径 | 是 | `admin` | admin |
| ENABLE_AUTH | 访客验证（`true` 开启，不设置或 `false` 关闭） | 否 | 关闭 | false |
| MAX_SIZE_MB | 单文件最大支持大小（单位：MB） | 否 | 20 | 20 |

> `ADMIN_PATH` 建议改成随机长串：管理后台是无条件 Basic Auth，路径本身即一层防护。

### 绑定说明

- **D1 数据库**：绑定变量名需与 `DATABASE` 环境变量一致

### 数据库结构

需要一张 `media` 表：

```sql
CREATE TABLE media (
    url TEXT PRIMARY KEY,
    fileId TEXT NOT NULL
);
```

`url` 字段的取值形态：

- **新格式**：裸 key，如 `1734567890123-a1b2c3d4.jpg`
- **旧格式**：完整 URL，如 `https://example.com/1734567890123.jpg`

两种形态都能被正确读取与删除，可永久共存。

---

## 数据形态统一（可选，在 D1 控制台执行）

**先说结论：这一步完全可以不做的。** 读取侧三种形态都能匹配，换域名也已彻底修好，
下面这些 SQL 纯粹是让库里的数据看起来更整齐。嫌麻烦直接跳过，不影响任何功能。

### 为什么是 SQL 而不是内置功能

- 项目里曾经内置过迁移接口，后来移除了 —— 一次性的数据整理，不值得长期占着一个 HTTP 端点、一份环境变量和一套前端面板。
- 在 D1 控制台执行**不消耗 Workers 请求数**，也不受免费计划 10 ms CPU 限制，比走 Worker 更省。

### 为什么必须分批、为什么是 1000 行

- **D1 单条 SQL 最长执行 30 秒**。Cloudflare 官方明确要求：影响百万行的 `UPDATE`/`DELETE` 必须分批，
  建议每批 **1000 行以内**。一条 `UPDATE` 直接改 100 万行必然超时或被拒绝。
- 每改 1 行 ≈ **2 rows written**（表行 1 次 + 主键索引 1 次）。D1 免费计划写入额度只有 **10 万/日**，
  1000 行就占掉 2000 次。分批可以随时观察消耗、随时停手，不至于一次性把当天额度打满导致 D1 拒绝查询。

### 准备工作：先看一眼

```sql
-- 1. 还剩多少行是旧的完整 URL 形态
SELECT COUNT(*) AS pending FROM media WHERE url >= 'http';

-- 2. 随机看几条，确认前缀长什么样
SELECT url FROM media WHERE url >= 'http' LIMIT 5;
```

> 这里用 `url >= 'http'` 而不是 `LIKE '%/%'`：裸 key 以数字开头（ASCII 里 `0`~`9` 都小于 `h`），
> 所以这个范围条件能精确圈出「非裸 key」的行，并且**走主键索引**；
> 而 `LIKE '%/%'` 用不上索引，每批都要全表扫一遍。

### 执行：一次改 1000 行

把 `https://旧域名/` 换成第 2 步看到的实际前缀，**反复执行下面这条，直到返回的影响行数为 0**：

```sql
UPDATE media
SET url = REPLACE(url, 'https://旧域名/', '')
WHERE rowid IN (
  SELECT rowid FROM media WHERE url >= 'http' LIMIT 1000
);
```

已实测验证：每条正好影响 1000 行，重复执行逐批推进（1000→999→…→0），
已改过的行不会被重复匹配（`REPLACE` 后 url 不再 `>= 'http'`），可以放心多跑几次。

> **关于 `rowid IN (SELECT ...)`**：SQLite 的 `UPDATE` 默认不支持 `LIMIT`
> （需要 `SQLITE_ENABLE_UPDATE_DELETE_LIMIT` 编译选项，D1 未开启），
> 所以先用子查询圈出 1000 个 `rowid`，再按主键定位更新 —— 定位是 O(1)，语句也最简。

**不确定旧域名、或库里混杂了多个旧域名**，用这条通用写法（取最后一个 `/` 之后的内容，无需填域名）：

```sql
UPDATE media
SET url = replace(url, rtrim(url, replace(url, '/', '')), '')
WHERE rowid IN (
  SELECT rowid FROM media WHERE url LIKE 'https://%' LIMIT 1000
);
```

### 收尾：验证

```sql
-- 应为 0
SELECT COUNT(*) FROM media WHERE url >= 'http';

-- 应等于迁移前的总数（一条都没丢）
SELECT COUNT(*) FROM media;
```

### 注意事项

- **先备份**：Cloudflare 后台 D1 → 你的数据库 → Export，或用
  `npx wrangler d1 export <数据库名> --remote --output=backup.sql`
- **`REPLACE` 的域名别填错**：填错时 `REPLACE` 匹配不到、值不变，但 `WHERE` 仍会命中，
  表现为「每次都改 1000 行却永远改不完」。执行一次后跑一下上面的验证语句，确认 `pending` 在下降。
- **同名冲突**：旧格式是纯时间戳，同毫秒同后缀才会撞主键，概率极低；真撞了会报 `UNIQUE constraint failed`，
  该批整批回滚，手动给冲突行换个后缀再跑即可。
- **百万行的性价比**：200 万 rows written 在免费计划下要跑 20 天，期间每天都撞额度上限。
  真有百万行，要么干脆不迁，要么开一个月 Workers Paid（含 5000 万 rows written/月，200 万只占 4%，**不产生超额费用**）再迁，约 1–3 小时跑完。

---

## 上传白名单说明

- 白名单与 `Content-Type` 映射共用一张 `CONTENT_TYPE_MAP`，新增格式只需在对应分类加一行
- **禁止**加入 `html / htm / js / mjs / css / xml / json / wasm / php` —— 会被同源托管为脚本，造成存储型 XSS
- `svg` 虽在白名单内，但强制以附件下载 + 严格 CSP，不会被浏览器内联渲染
- 所有响应带 `X-Content-Type-Options: nosniff`，防止 MIME 嗅探
- 音频、压缩包、文档在管理页显示 📁 图标（无预览），不影响正常下载

---

## 更新日志

### 本 Fork

#### v2.2.0
- 管理页改为**游标分页**（`WHERE url < ?`），修复 `OFFSET` 深翻页扫描近百万行的问题
- 支持双向游标导航 + 「回到最新」，每页恒定只读 51 行
- 实测 30 万行库深翻页提速约 100 倍，结果与 OFFSET 逐行一致

#### v2.1.1
- 新增 `APP_VERSION` 版本号，首页与管理页显示版本徽章
- 数据库改为存储裸 key，与域名彻底解耦；读取时兼容三种 URL 形态
- 管理页 `COUNT(*)` 增加 60 秒 isolate 级缓存

#### v2.1.0
- 上传白名单扩充至 61 项，新增音频、压缩包、文档（pdf/txt/docx 等）分类
- 强制下载后缀抽成 `ATTACHMENT_EXTENSIONS`（当前含 `svg`）
- 修复管理页 `<source type>` 使用非法 MIME 导致视频不播放的问题
- 首页缓存键规范化，多域名共用一份缓存

#### v2.0.0
- 修复换域名后历史文件全部 404 的问题（规范化 URL 查询）
- `ADMIN_PATH` 增加默认值 `admin`

### 上游历史（2026-09-13 及之前）

<details>
<summary>展开查看上游更新记录</summary>

#### 2026-09-13
- UI 重构为苹果液态玻璃（Liquid Glass）风格
- 卡片新增鼠标跟随镜面高光效果
- 通知系统重构为右上角玻璃 Toast
- 管理页危险操作改用玻璃风格 `glassConfirm()` 弹窗
- 首页与管理后台样式独立打包

#### 2026-09-12
- 移除首页 jQuery / Bootstrap / Fileinput / FontAwesome / toastr 外部依赖，首屏 220 KB → 12 KB
- 上传 key 增加随机后缀，修复同毫秒并发上传互相覆盖
- 上传扩展名白名单校验，阻止 `.html` / `.js` 上传
- 上传鉴权前置，未授权请求不再消耗 Telegram API 配额
- 删除接口校验 URL 归属
- SVG 强制以附件形式下载 + 严格 CSP 防止存储型 XSS
- Telegram `getFile` 失败带退避重试（200ms / 400ms）
- 路由归一化，图片缓存键去掉 query 参数，JSON 响应加 `no-store`
- 修复缓存列表顺序被永久修改、上传成功提示被清除、同一文件二次选择不触发上传等问题
- 支持一次粘贴多个文件，顶层异常统一返回 JSON

#### 2026-01-19
- 使用 Claude 优化了一下代码

#### 2025-08-24
- 修复 cdn.bytedance.com 下线导致的页面加载异常

#### 2025-08-07
- 修复主页背景图片无法加载的问题

#### 2024-12-18
- 更新管理界面样式，移除前端文件类型和大小限制，改由环境变量控制

#### 2024-12-17
- 前端新增压缩按钮，默认开启

#### 2024-12-13
- 哈希校验避免重复上传，压缩率 0.75，删除接口添加认证

#### 2024-11-29
- 管理页新增全选和复制，删除二次确认，禁用视频自动播放
- 修复粘贴上传时不显示移除按钮

#### 2024-11-21
- 优化上传体验，默认开启压缩

#### 2024-11-01
- 修复上传后无法加载的问题

#### 2024-10-19
- 修复 webp 无法上传的 BUG，优化数据库结构

#### 2024-09-29
- 优化缓存功能，采用 Cloudflare Cache API

#### 2024-09-25
- 修复 GIF 上传问题；Telegraph 接口移到 telegraph 分支，main 分支为 TG_BOT 接口

#### 2024-09-23
- 修复链接失效问题，支持视频上传

#### 2024-09-14
- Telegraph 接口上传的文件有时效性，建议使用 TG_BOT 上传

#### 2024-09-13
- 支持通过 TG_BOT 上传到频道

#### 2024-09-06
- telegra.ph 禁止上传媒体文件，原项目终结

</details>

---

## 从上游升级

1. 用本仓库的 `_worker.js` 整份替换
2. 环境变量无需任何改动
3. **Purge Everything** 清一次缓存（首页 HTML 缓存 3600 秒，不清可能看不到新版本号）
4. 老链接随便点几条验证，应照常打开

数据库表结构与上游完全一致，旧数据可直接沿用，无需迁移。

---

## 许可

本仓库继承上游仓库的 LICENSE，使用 MIT 协议。
