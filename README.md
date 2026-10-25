# Telegraph 图床

基于 Cloudflare Worker 和 Telegram Bot API 的图床 / 视频床 / 文件床服务

> 本仓库 fork 自 [0-RTT/telegraph](https://github.com/0-RTT/telegraph)（上游基线：2026-09-13），
> 在此之上做了独立维护与增强。
> **升级注意：v2.3.0 更改了默认认证行为，不属于完全无缝的行为兼容升级。** 部署前请配置强 `USERNAME` / `PASSWORD`；只有显式设置 `DISABLE_AUTH=true` 才会关闭首页与上传认证。跨域名使用的历史完整 URL 记录，也需按下文说明迁移。
> 当前版本：**v2.3.0**（页面底部与管理页头部会显示版本号徽章）

---

## 本 Fork 相比上游的改动

### 1. 数据与域名彻底解耦（换域名不再 404）

上游把完整 URL（`https://example.com/1734567890123.jpg`）存进 D1 且作为主键，
一旦更换域名，查询字符串对不上，历史文件**全部 404**。本 Fork 的修复分两层：

- **存储层**：新上传只存「裸 key」`1734567890123-a1b2c3d4.jpg`，域名不再入库。
- **读取层**：使用一次索引查询匹配裸 key、配置域名 URL、当前请求域名 URL。
- **重要兼容边界**：未迁移的旧记录仍保存原主机名。只有旧主机名恰好等于配置域名或当前请求主机时才会命中；切换到全新域名前，必须按下方 SQL 将历史 URL 迁移成裸 key。
- 缓存键统一为规范化 URL，避免不同入口对同一裸 key 使用不同缓存键。
- 删除接口支持裸 key 和当前配置域名下的完整 URL；跨域名历史 URL 需要迁移后才能稳定删除。

> 该版本不再声称任意历史域名的完整 URL 可以永久免迁移共存。迁移仅需改 D1 中的 key，不重新上传 Telegram 文件，也不改变 fileId。

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
- `svg` 强制 `Content-Disposition: attachment` + 严格 CSP；`pdf` 同样强制附件下载，避免同源内联文档渲染
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
- 🔐 默认启用访客验证（Basic Auth；仅 DISABLE_AUTH=true 关闭）
- 🗜️ 可选的图片压缩功能（默认开启，支持前端切换）
- 📦 可选的文件大小限制（默认 20MB，可通过环境变量配置）
- 📁 支持图片、视频等常见媒体格式上传（白名单校验，防止存储型 XSS）
- 📤 支持多文件上传、拖拽上传和粘贴上传（Ctrl+V）
- 🔄 哈希校验避免重复上传
- 🚀 多文件并发上传（默认 4 路并发）

### 管理功能
- 📋 支持查看本地历史记录
- 🖼️ 图库管理界面，支持批量操作
- 🗑️ 支持批量删除索引（删除 D1 记录并尝试清理当前 Worker 缓存；不删除 Telegram 原始文件，也不能立即撤销全球/浏览器缓存）
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
- ⚡ Cloudflare Cache API 缓存支持（仅缓存完整 200 响应；Range 请求透传且不写共享缓存）
- 🎨 懒加载和骨架屏优化
- 🌅 Bing 每日壁纸背景（自动轮播）
- 🔁 获取文件路径自动重试（最多 3 次，带退避）
- 🪶 首页与管理页均零外部依赖（无 jQuery / Bootstrap / FontAwesome / toastr / 任何 CDN 资源）

### 存储方式
- 📡 基于 Telegram Bot API 的文件存储
- 💾 使用 Cloudflare D1 数据库存储文件映射关系
- 🎯 通过 fileId 实现文件访问
- 🔗 新上传的 url 以裸 key 形式存储；旧的完整 URL 记录需迁移后才能彻底解耦域名

---

## 部署

> ⚠️ 默认启用首页与上传接口的 Basic Auth 防护。只有设置 `DISABLE_AUTH=true` 才会关闭这两处防护；媒体文件 URL 仍为公开访问。删除操作只删除 D1 索引，不会删除 Telegram 频道中的原始消息，也无法强制清空全球边缘或浏览器缓存。

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
| DISABLE_AUTH | 关闭首页与上传接口的 Basic Auth（仅严格等于 `true` 时关闭） | 否 | `false`（防护开启） | true |
| MAX_SIZE_MB | 单文件最大支持大小（单位：MB） | 否 | 20 | 20 |

> `USERNAME` 和 `PASSWORD` 必须配置为强凭据，否则默认保护的首页与上传接口会返回 401。管理后台和删除接口始终要求管理员认证。旧变量 `ENABLE_AUTH` 将被忽略；只有 `DISABLE_AUTH=true` 才关闭首页与上传接口的访客认证。媒体文件 URL 仍是公开的。
>
> `ADMIN_PATH` 建议改成随机长串，但它只能减少被扫描发现的概率，不能替代 Basic Auth。

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

## 旧数据迁移（跨域名切换前必做）

新上传记录保存的是裸 key，例如 1734567890123-a1b2c3d4.jpg。旧版本记录则可能是完整 URL，例如 https://old.example.com/1734567890123.jpg。代码为避免每次读取都全表扫描，只会查询裸 key、当前配置域名 URL 和当前请求域名 URL；因此，切换到一个全新的主机名时，未迁移的旧记录可能无法访问或删除。

迁移只修改 D1 的主键值，不会重新上传文件，也不会改变 Telegram 的 fileId。**先备份 D1，再执行以下步骤。**

### 1. 检查待迁移数据

    SELECT COUNT(*) AS legacy_count
    FROM media
    WHERE url LIKE 'http%://%/%';

    SELECT url
    FROM media
    WHERE url LIKE 'http%://%/%'
    LIMIT 10;

### 2. 检查迁移后可能发生的主键冲突

以下查询应返回 **0 行**。若有结果，请先人工处理重复 key；不要直接执行 UPDATE，否则主键冲突会导致对应批次失败。

    -- 多条历史 URL 归一化后得到相同 key
    WITH legacy AS (
      SELECT url,
             substr(
               url,
               instr(url, '://') + 3
                 + instr(substr(url, instr(url, '://') + 3), '/')
             ) AS bare_key
      FROM media
      WHERE url LIKE 'http%://%/%'
    )
    SELECT bare_key, COUNT(*) AS n
    FROM legacy
    GROUP BY bare_key
    HAVING COUNT(*) > 1;

    -- 历史 URL 归一化后会覆盖已经存在的裸 key
    WITH legacy AS (
      SELECT url,
             substr(
               url,
               instr(url, '://') + 3
                 + instr(substr(url, instr(url, '://') + 3), '/')
             ) AS bare_key
      FROM media
      WHERE url LIKE 'http%://%/%'
    )
    SELECT legacy.url AS legacy_url, legacy.bare_key
    FROM legacy
    JOIN media AS existing ON existing.url = legacy.bare_key
    WHERE existing.url <> legacy.url;

### 3. 分批迁移，每批最多 1000 行

重复执行以下语句，直到影响行数为 0。更新后的 key 不再匹配完整 URL 条件，因此后续批次会继续处理剩余记录。

    UPDATE media
    SET url = substr(
      url,
      instr(url, '://') + 3
        + instr(substr(url, instr(url, '://') + 3), '/')
    )
    WHERE rowid IN (
      SELECT rowid
      FROM media
      WHERE url LIKE 'http%://%/%'
      LIMIT 1000
    );

### 4. 验证

    -- 应返回 0
    SELECT COUNT(*) FROM media WHERE url LIKE 'http%://%/%';

    -- 行数应与迁移前一致
    SELECT COUNT(*) FROM media;

> 注意：迁移前先确认所有历史 URL 都是“主机名 + 单个文件 key”结构。遇到不符合此结构的自定义数据，应单独处理。迁移后，新旧主键将统一成裸 key，游标分页的字典序也能按时间戳前缀自然排列。

## 上传白名单说明

- 白名单与 `Content-Type` 映射共用一张 `CONTENT_TYPE_MAP`，新增格式只需在对应分类加一行
- **禁止**加入 `html / htm / js / mjs / css / xml / json / wasm / php` —— 会被同源托管为脚本，造成存储型 XSS
- `svg` 虽在白名单内，但强制以附件下载 + 严格 CSP，不会被浏览器内联渲染
- 所有响应带 `X-Content-Type-Options: nosniff`，防止 MIME 嗅探
- 音频、压缩包、文档在管理页显示 📁 图标（无预览），不影响正常下载

---

## 更新日志

### 本 Fork

#### v2.3.0
- 默认开启首页与上传接口 Basic Auth 防护；仅 `DISABLE_AUTH=true` 可关闭，忽略旧 `ENABLE_AUTH`
- 修复空文件 key 与根路径缓存键碰撞；拒绝嵌套/尾斜杠的非文件路径
- 加入上传/删除写接口的来源与 Content-Type 校验，增加请求体及批量删除上限
- 校验 `MAX_SIZE_MB`，无效配置回退 20 MB；上传前提前检查 Content-Length
- 支持透传 Range 请求；不缓存 206/Range 响应，重建文件响应头白名单
- PDF 强制附件下载；GIF 保留真实文件名与类型
- 修正文档：旧域名数据跨域切换前须迁移；明确删除索引不会删除 Telegram 原始文件

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
2. 配置强 `USERNAME` / `PASSWORD`；`ENABLE_AUTH` 不再生效，只有 `DISABLE_AUTH=true` 才关闭首页与上传认证
3. **Purge Everything** 清一次缓存（首页 HTML 缓存 3600 秒，不清可能看不到新版本号）
4. 老链接随便点几条验证，应照常打开

数据库表结构与上游完全一致，但如果旧记录的完整 URL 使用的主机名不同于当前 `DOMAIN`，必须先执行上方的 D1 迁移步骤。升级前请确认已配置强 `USERNAME` / `PASSWORD`；默认情况下首页与上传接口将要求认证。

---

## 许可

本仓库继承上游仓库的 LICENSE，使用 MIT 协议。
