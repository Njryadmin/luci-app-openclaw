# Issue Triage 2026-08-13

本文件记录 2026-08-13 维护周期（PR #101 / #102 / #103 + fork 化迁移）相关 issue / PR 的处理边界，便于后续关闭 issue 时引用。
本周期变更已合并进 fork `Njryadmin/luci-app-openclaw` 的 `main` 分支，等待向上游 `10000ge10000/luci-app-openclaw` 推送。

## 已在 2.1.0 覆盖

### PR #101 — Makefile feeds 检测修复（+14/-27）

- 修复在 OpenWrt SDK 单包编译环境下，`feeds.conf.default` 中没有 `src-git openclaw ...` 条目时，`package/luci-app-openclaw/` 直接放源码也能正确构建。
- 不再强依赖 feeds 注入；`Makefile` 顶层 `include $(TOPDIR)/rules.mk` 与 `$(INCLUDE_DIR)/package.mk` 链路已对齐 SDK 单包目录结构。
- 顺带清理 `Makefile` 中重复的 `PKG_NAME/PKG_VERSION` 引用，统一从 `VERSION` 文件读取版本号。

### PR #102 — Telegram 配对修复（+26/-4）

- 解决 issue #98 报告的 Telegram 配对流程在某些网络环境下卡在「等待配对」状态。
- 配对 API 调用增加可配置超时和重试，配对状态轮询改用前台增量日志，避免被后端 504 拖死。
- 失败信息统一带上 `chat_id` / `bot_username` 上下文，方便用户贴日志时排错。

## 已在 2.2.0 覆盖（PR #103，待合入）

### 模型供应商管理（LuCI-native 替代 Web PTY）

- 之前添加/切换/测试模型只能进 Web PTY（端口 18793）跑 `oc-config` 交互式 CLI，本 PR 改为 LuCI 子菜单可视化操作。
- 新增独立子菜单 `服务 → OpenClaw → 模型管理`（路由 `admin/services/openclaw/models`），与「Web 控制台」「微信配置」平级。
- 6 个新路由：`models` (GET 渲染) / `models_api` (GET JSON) / `models_save` / `models_delete` / `models_test` / `models_set_active`，均仅做 `formvalue` 解析 + 文件读写 + 异步 gateway 重载，权限收敛在 `luci-app-openclaw` ACL 之内。
- 6 个新 helper（均 `local` 闭包，无全局副作用）：
  - `read_openclaw_config` — 经 `luci.jsonc.parse` 读 `data/.openclaw/openclaw.json`，失败回退到 `config_not_found`。
  - `write_openclaw_config` — 备份 `.bak-<unix_ts>` → 序列化 → 写回 → `chown openclaw:openclaw` → `chmod 600` → 调 `openclaw-permissions.sh fix-state` → 异步触发 `restart_gateway`（SIGUSR1 in-process reload）。
  - `mask_api_key` — 头 4 + `••••` + 尾 4，长度 <12 时退化为 `****`。
  - `test_connectivity` — 5s `curl -sI --connect-timeout 5 --max-time 8` 连通性预检。
  - `test_provider_api` — OpenAI 兼容走 `GET /v1/models`，Anthropic 走最小 payload `POST /v1/messages` (`claude-3-5-haiku-20241022` + `max_tokens: 1`)。
  - `MODEL_PROVIDER_PRESETS` — 16 个预设，**国内优先**（qwen / bailian / lkeap / siliconflow / baidu / zhipu / yiwanai-fan 排前），方便国内用户开箱即用。
- 写配置后**异步**触发 `/etc/init.d/openclaw restart_gateway`（SIGUSR1 in-process restart，~1-2s 完成），新模型/活跃模型对运行中的 gateway 立即生效，无需手动重启服务。`restart_gateway` 内部 `lock_dir` 保护并发安全；不影响 Web PTY。
- 防御性约束：删除供应商时若 `agents.defaults.model.primary` 来自该 provider，返回 `provider_is_active` 错误并要求先切换活跃；添加第一个 provider 时若没有活跃模型，自动设为 `primary`。
- ID 校验：`provider` / `modelId` 必须匹配 `^[%w][%w\-_\.]*$`（避免路径遍历 / 注入）。
- 视图模板 `luasrc/view/openclaw/models.htm`（876 行）：卡片网格 + 增改 Modal + XHR 驱动；服务端预渲染一次，XHR `models_api` 立即覆盖保证新鲜；空状态、未安装降级、活跃标记、删除二次确认、Toast 反馈完整。

## Fork 化迁移（2026-08 上旬）

为方便 PR 流程，fork `Njryadmin/luci-app-openclaw` 的 `main` 分支相对上游 `10000ge10000/luci-app-openclaw` 做了以下元数据重定向（不影响行为）：

- `.github/workflows/build.yml`、`Makefile`、`scripts/build_ipk.sh`、`scripts/gen-release-body.sh` 中的 owner / repo 引用改为 `Njryadmin`。
- `luasrc/controller/openclaw.lua`、`root/usr/bin/openclaw-env`、`luasrc/model/cbi/openclaw/basic.lua`、`README.md` 中涉及仓库地址的文案同步更新。
- 后续通过 fork `main` → 上游 `main` 的 PR 推回。

## 已由当前 main 既有实现覆盖

- Web PTY 自动 respawn：procd 检测 `web-pty.js` 进程退出后会立即拉起，热更场景下 `~1-2s` 抖动是设计内行为。
- `openclaw.json` 备份链路：`write_openclaw_config` 沿用 `data/.openclaw/openclaw.json.bak-<unix_ts>` 时间戳备份，**不**使用单一滚动 `.bak`，避免并发写时互相覆盖。
- `sync_uci_to_json` 在 `start_service` 时跑，把 UCI `openclaw.main.*` 端口 / token / bind / install_path 同步进 JSON；JSON → UCI 方向在 token 不一致时以 JSON 为准。
- 路由表自检：6 个新 `entry()` 全部追加在 `index()` 末尾，**不动**既有 `entry` 的 `order`、依赖或 `leaf` 标记；新页面在 LuCI 菜单中的位置由 `entry(... 11)` 控制（基本设置=10 之后）。

## 暂不关闭

以下类型 issue 不应仅凭 2.1.0 / 2.2.0 关闭，需要单独复现或确认：

- 模型/provider 具体行为差异或上游 OpenClaw 功能请求。
- 涉及 Web PTY（端口 18793）无法登录、token 错误、iframe 嵌入异常的问题——本 PR 不动 PTY 流程，关闭前应先在「Web 控制台」页确认 gateway 是否能正常监听 18789。
- 需要用户确认网络、DNS、代理、设备资源或第三方 API 状态的问题。
- 涉及 OpenClaw 网关本身启动失败（`exit code 1` 等）的 issue：v2.2.0 只动「配置面」，不动「运行面」；如已在路由器上观察到 gateway 启动失败，应回到「服务控制」先 `openclaw-env doctor` 排错。
- 涉及 LuCI 缓存命中老版本视图导致的显示异常：清 LuCI 缓存 (`rm -f /tmp/luci-indexcache /tmp/luci-modulecache/*`) 或硬刷浏览器（Ctrl+Shift+R）后再判断。

## 升级建议

v2.2.0 在热更场景下行为：

- Web PTY 端口 18793 断 ~1-2s（procd 自动 respawn），正在进行中的 `oc-config` 配对/操作会丢。
- Gateway 进程 18789 不动，内存中模型/渠道配置**完整保留**。
- UCI `openclaw.main.*`（port / bind / token / install_path）通过 `install.sh:154-160` 智能保留，**不会**回退到默认。
- `/mnt/sda1/openclaw/data/.openclaw/openclaw.json` 不被动。
- 新页面（v2.2.0）第一次访问建议硬刷一次浏览器，避免 JS 缓存命中老模板。
