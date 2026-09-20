# Rikkahub for LazyCat (rikkahub-lzcapp)

[Rikkahub Desktop](https://github.com/yuh-G/rikkahub-desktop) 打包为懒猫微服 LPK v2 应用：多供应商 LLM 聊天客户端，AGPL-3.0（非商业/个人用途）。

## 包信息

- 包名：`community.lazycat.app.rikkahub`（1.5.0）
- 上游镜像：`ghcr.io/yuh-g/rikkahub-desktop:v1.5.0`
- 交付方式：**mirror**（`ghcr.1ms.run` 加速器，digest 校验），仅发布**喵喵私有商店**
- 目标架构：amd64；`min_os_version: 1.5.0`

## 转换要点

| docker run | LPK |
|---|---|
| `-p 8080:8080` | `upstreams: / → http://rikkahub:8080/` |
| `-v ./pc-data:/app/pc-data` | `binds: /lzcapp/var/pc-data:/app/pc-data` |
| 无 | `RIKKAHUB_PASSWORD` + 安装向导密码 + inject 自动填充 |

`command` 不需要覆盖：镜像的 ENTRYPOINT 已经是 `./rikkahub-pc --no-open`。

## 三个必须处理的坑

**1. 镜像基于 distroless，没有 shell、没有 curl。**
所以不能像常规转换那样用 `command: sh ...` 包一层启动脚本，也不能写 `healthcheck: curl ...`。本包直接用镜像自带 ENTRYPOINT，并且**不添加 healthcheck**（上游 docker run 本来就没有；容器内也确实没有任何可用探测命令）。

**2. 不显式声明容器环境，应用只会监听 127.0.0.1。**
`pc-server/foundation/platform.ts` 用 `existsSync("/.dockerenv")` 判断是否在容器里，而懒猫的运行时不会创建这个 Docker 专有标记。判断为 false 时应用会走「桌面形态」：绑定 `127.0.0.1`、并且会去读 state.json 里的用户端口设置。所以显式设了：

```yaml
- RIKKAHUB_CONTAINER=1   # 绑定 0.0.0.0、端口固定 8080、禁用容器内自更新
- RIKKAHUB_HOST=0.0.0.0
```

**3. 应用有 Origin/Host 一致的 CSRF 校验。**
`server.ts` 的 `isAllowedOrigin()` 对 `/api/*` 做检查：没有 Origin 头 → 放行；Origin 与请求 Host 完全相同 → 放行；其余 403。上游 README 专门警告过反代改写 Host 会导致「网页打得开但什么都保存不了」。

懒猫网关的 `upstreams` 默认 `use_backend_host: false`，即**转发浏览器原始 Host**，所以 Origin 与 Host 天然一致，不需要额外配置。如果实测出现 POST/DELETE 全部 403，说明网关改写了 Host，两种修法：

- 给 upstream 加 `remove_this_request_headers: [Origin]`（会同时失去应用的 CSRF 防护）
- 或确认网关是否支持透传 Host

## 鉴权

应用默认**零鉴权**（它假设你只监听本机）。本包按选择启用自带密码闸门：

- `RIKKAHUB_PASSWORD` 来自安装向导参数，token 用 HMAC-SHA256 无状态签名，有效期 30 天
- 前端在任意 `/api/*` 收到 401 时弹出密码闸门（SPA 组件，无独立路由，页面始终在 `/`）
- 闸门只有一个密码框、没有用户名，所以 inject 开了 `allowPasswordOnly: true`
- `public_path: /`：关闭微服网关的 HTTP 账号鉴权，访问仍限虚拟网络内；鉴权由应用自身承担

不想要密码的话，把安装参数留空即可（不推荐，应用内保存着 API Key 与全部会话）。

## 部署参数

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `login_password` | secret | `$random(len=20)` | 访问闸门密码，留空则不启用鉴权 |

## 本地构建与验证

```sh
lzc-cli project release
lzc-cli lpk info dist/*.lpk
```

## 自动发布（仅喵喵商店）

`.github/workflows/lazycat.yml` 每日 06:41 UTC（或手动触发）：

1. 检查 `ghcr.io/yuh-g/rikkahub-desktop` 新的 `vX.Y.Z` tag（`tag_regex: ^v?\d+\.\d+\.\d+$`，semver 排序）
2. 校验 `ghcr.1ms.run` 镜像 digest 与源一致（`require_digest_match: true`），更新 manifest 与版本
3. 构建 LPK → 提交 → 打 tag → GitHub Release（`community.lazycat.app.rikkahub-v<version>.lpk`）
4. 发布到喵喵私有商店

所需 Secrets：`APPSTORE_URL`、`APPSTORE_TOKEN`（必填），`APP_ID`、`PRIVATE_STORE_GROUP_CODES`（可选）。组织级已配置。

`v2.0.0-preview` / `latest` / `main` / `worktree-*` 不属于发布族，不会被选中。

## 已知限制

- 容器内没有剪贴板工具和 espeak-ng，**复制按钮与系统 TTS 不可用**；上游 README 已说明这一点，其余功能不受影响
- 未接入懒猫文件选择器拦截（按需求）：上传附件、导入 `.zip` 备份走浏览器原生文件选择
- 应用的 `scrape_web` 工具会在容器内直接抓取模型给出的任意 URL，存在 SSRF 风险，上游 README 亦有提示
- 图标取自上游 `icons/rikkahub.svg`（矢量重绘为 512×512），仓库内的 `docs/icon.png` 只有 192×192
