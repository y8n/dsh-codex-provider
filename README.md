# dsh-codex-provider

> [!IMPORTANT]
> **这是 [Hu9956/dsh-codex-provider](https://github.com/Hu9956/dsh-codex-provider) 的 fork，只为把它跑在 DSH `0.2.0-rc.2` 上。**
>
> 上游已于 2026-08-13 停止维护，并且它的 `peerDependencies` 锁在 `^0.1.0-rc.6`，
> 在 DSH 0.2 上安装时会被插件管理器直接拒绝：
>
> ```
> Plugin dsh-codex-provider@0.1.0 is incompatible with dsh 0.2.0-rc.2
> ```
>
> 本 fork 的改动（均在 `lib/` 与 `package.json`）：
>
> | 改动 | 原因 |
> |---|---|
> | `peerDependencies` 全部改为 `^0.2.0-rc.2`，删掉 0.2 已不存在的 `@deepseek-ai/dsh-client-runtime`、`@deepseek-ai/dsh-client-web-react` | 版本栅栏按运行时版本逐条比对 peer 范围，旧范围必然被拒 |
> | client 侧 Remote 描述符的 `result` 从 `{ mode: "strict", typeSymbol, schema }` 改为 `{ mode: "src-json" }` | **0.2 的 `typert.remotes.register()` 只接受 `{mode:"strict", typeSymbol, create()}` 或 `{mode:"src-json"}`；旧写法抛 `strict codec has no create() factory`** |
> | `apply()` 里的 `$mount` / UI 装配全部 try/catch，失败只 `console.error` 不抛出 | DSH 的 web boot 审计把「client entry 失败」当致命错误，恢复流程会**禁用 profile 里全部第三方 bundle**（见下） |
> | `settings.get(ns)` → `settings.describe()` | DSH 0.2 的 settings 缝只剩 `describe()/update()/replace()/mutate()`，`get()` 已移除 |
> | `<Pill tone>` → `<Pill active>` | 0.2 的 primitives `Pill` 只有 `{ active, className, onClick }` |
> | `Button variant="secondary"/"danger"` → `"outline"` + 危险色内联样式 | 0.2 的 `Button` 只认 `ghost / outline / primary` |
> | 主题变量改用 `--dsw-alias-state-error-primary` / `--dsw-alias-state-success-primary` | 旧变量在新主题里已不是规范名 |
>
> ### 一次真实事故（2026-10-02）
>
> 只改 peer 范围、没改描述符就装上 DSH 桌面端 0.2.0-rc.2，重启后应用直接进
> **致命恢复流程**：`~/Library/Logs/DeepSeek Harness/crash-*.log` 记录
> `web boot: 1 entry did not activate / dsh-codex-provider: failed`，
> 恢复动作把 `dsh.profile.bundles` 清成只剩 `dsh-base` + `dsh-web-app`，
> 并把用户自己的 `cordis.patch.yml` 备份后替换成模板 —— **该 profile 下所有第三方插件一起失效**。
>
> 根因就在上面第二行：`DescriptorStore.validate → validateInvocation → validateCodec`。
> 这个 fork 除了修 codec，还给 `apply()` 加了兜底，让同类问题退化为「设置里看不到分区」
> 而不是整机插件团灭。改动经过 DSH 0.2.0-rc.2 真实 `remote.$mount()` 代码路径验证。
>
> 除此之外的 OAuth、凭证、令牌刷新逻辑与上游一致。
>
> 上游推荐的替代方案 [Magpie](https://github.com/yetone/magpie) 仍然可用，且不需要改配置文件；
> 如果你更想要一个装好即用的本地网关，请看上游 README 的存档段落。

DeepSeek Harness (DSH) 供应商插件：**OpenAI Codex（ChatGPT Plus/Pro 订阅）** 的设备码 OAuth 登录、令牌自动刷新与供应商管理。

在 DSH 设置中新增 **“供应商”** 分区，通过设备码授权登录 OpenAI 账号后，即可在模型选择器中使用 DSH 内置的 `openai-codex` 模型（`gpt-5.4`、`gpt-5.5`、`gpt-5.6-*` 等），消费你的 **ChatGPT Plus / Pro 订阅额度**。

## ✨ 功能

- **设备码 OAuth 登录**：在界面点击“使用 OpenAI 账号登录”，按提示完成设备码授权，无需 API Key
- **导入 Codex CLI 登录态**：直接复用本机 `~/.codex/auth.json` 中已有的登录（无需重新授权）
- **令牌自动刷新**：后台定期检查 access token，过期前自动用 refresh token 轮换（refresh_token 也会一并轮换）
- **供应商管理界面**：设置 → 供应商，展示登录状态、账号、套餐（Plus/Pro）、令牌过期时间，支持退出登录
- **激活内置 Codex 模型**：登录后 `openai-codex` 路由自动激活，模型选择器直接可选


<img width="1590" height="776" alt="ScreenShot_2026-08-14_020002_683_副本" src="https://github.com/user-attachments/assets/4c2d8b0e-edb8-46a8-876d-f297f0ec8f54" />
<img width="1892" height="1020" alt="ScreenShot_2026-08-14_043716_413" src="https://github.com/user-attachments/assets/09abd005-4432-4eb9-b9fc-6acab9772361" />


## 📦 安装

从本 fork 的 GitHub 源码安装（推荐，`desktop` 是 DSH 桌面端使用的 profile）：

```bash
dsh plugin --profile desktop add github:y8n/dsh-codex-provider
```

也可以本地目录方式安装（`pnpm` 会建立链接，改完代码重启即可生效）：

```bash
dsh plugin --profile desktop add /绝对路径/dsh-codex-provider
```

插件包自带 DSH bundle 配置，安装时会自动加入 profile，无需手动编辑 `cordis.patch.yml`。
桌面端也可以直接在 **设置 → 插件** 里用上面的 spec 安装。

安装后需要重启 DSH 桌面端（关闭并重新打开 App），刷新页面即可在 **设置 → 供应商** 看到入口。

> 注意：上游 npm 上的 `dsh-codex-provider@0.1.0` 仍是旧版本，peer 范围锁在 `^0.1.0-rc.6`，
> 在 DSH 0.2 上会被插件管理器判定为 `incompatible-version` 而拒绝安装。

## 🚀 使用

1. 打开 **设置 → 供应商**
2. 点击 **“使用 OpenAI 账号登录”**，按页面提示完成设备码授权
3. 登录成功后，在模型选择器中选择 Codex 模型（如 `gpt-5.4`）开始使用

### 设备码授权注意事项（页面也会显示）

1. **登录前**：请先在 **ChatGPT 网页端 → 设置 → 账户安全与登录** 中打开“为 Codex 启用设备代码授权”开关（未开启将无法授权）；
2. **授权**：先复制页面显示的设备代码，再点击“使用 OpenAI 账号登录”，按提示填入该设备代码；
3. **登录后**：建议返回 ChatGPT 网页端关闭该开关——不影响本次登录，但下次重新登录前需重新开启。

## ⚙️ 工作原理

- 设备码 OAuth 流程直接对接 `auth.openai.com`，令牌直连 `chatgpt.com/backend-api`（与官方 Codex CLI 相同的通道与认证方式）
- access token / refresh token 存入 DSH 凭证库（`OPENAI_CODEX_API_KEY` / `OPENAI_CODEX_REFRESH_TOKEN`）；仅 Host 侧在请求和刷新时解析，绝不返回浏览器
- 凭证引用写入 `llm-pi-ai.providers.openai-codex.apiKeyEnv`，激活 DSH 内置的 `openai-codex` 路由（`routeAuth` 会在 OAuth 旁附加 harness apiKey 通道）
- 后台任务每分钟检查 access token 过期时间，剩余不足 10 分钟时自动刷新并轮换；同机多实例通过刷新锁串行处理

## ⚠️ 注意事项

- **与 Codex CLI 共享同一 OAuth 会话**：任一侧刷新令牌后，另一侧持有的旧 refresh token 可能失效（届时在 Codex CLI 侧重新 `codex login` 即可）
- **不需要 API Key**：本插件消费的是 ChatGPT 订阅额度，不是 OpenAI API 计费
- 需要 DSH `>= 0.2.0-rc.2`
- 可用模型由当前 DSH 版本和 OpenAI 账户权限决定

## 🔒 安全与隐私

- access token 与 refresh token 只写入 DSH 凭证库；浏览器端只接收脱敏账号信息和登录状态
- 请勿在 Issue、Discussion、日志或截图中公开设备代码、OAuth token、`~/.dsh/.credentials.yaml` 或 `~/.codex/auth.json` 的内容
- 安全问题请按 [SECURITY.md](SECURITY.md) 通过 GitHub 私下报告

## 兼容性

当前版本针对 `@deepseek-ai/dsh 0.2.0-rc.2`（DSH 桌面端 0.2.0-rc.2 / Node.js 24 运行时）验证。
DSH 尚处于 RC 阶段，`0.2.x` 内的接口若再调整，本插件也需要同步升级；`0.1.x` 已不再作为目标版本
（`settings.get()` 在 0.2 被移除，代码里保留了回退分支，但 UI 与 peer 范围按 0.2 收敛）。

## 🛠️ 开发

| 文件 | 说明 |
|---|---|
| `lib/index.js` | host 插件：设备码 OAuth、凭证存取、令牌刷新、Typert Remote 服务（`codexProvider`） |
| `lib/client.js` | client 插件：设置页“供应商”分区 UI（ModuleLoader bundle 格式） |

## 📄 License

[MIT](LICENSE)

## 免责声明

这是社区维护的非官方插件，与 OpenAI、DeepSeek 或 DeepSeek Harness 官方没有隶属或背书关系。OpenAI、ChatGPT、Codex 和 DeepSeek 等名称与商标归各自权利人所有。
