# Custom Deploy（个人部署工作流）

本目录下的文件是**个人专属文件**，上游 `cmliu/edgetunnel` 不存在这些路径，因此
`git rebase` / 合并上游时不会产生任何冲突。

| 文件 | 作用 |
| --- | --- |
| `.github/custom-worker/wrangler-et.toml` | `cf-et` Worker 的部署配置 |
| `.github/workflows/custom-deploy.yml` | 手动触发并部署 `cf-et` 的 GitHub Actions 工作流 |

设计边界（已确认）：**Actions 只负责"部署 + 绑定"这类面板操作繁琐的事**；Worker 的内部运行变量
（`UUID` / `ADMIN` / `PROXYIP` 等）继续在 Cloudflare 面板维护，不进仓库、不暴露到 public、可随时调整。

## 一、需要的仓库 Secret

`Settings` → `Secrets and variables` → `Actions` → `New repository secret`

| Secret | 说明 |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API Token，权限至少需要 **Workers Scripts: Edit**（Account 级） |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare 账户 ID（面板 Workers 概览页右侧可复制） |

## 二、触发部署

`Actions` → 左侧 `Custom Deploy` → `Run workflow`

| 输入项 | 说明 |
| --- | --- |
| `ref` | 要部署的 Git 引用（分支 / 标签 / commit），留空 = 触发时所在引用 |
| `dry_run` | 勾选后只做编译校验，不上传部署 |

流程顺序：安装 wrangler → 校验 Secret → 打印账号信息 → dry-run 编译校验 → 正式部署 `cf-et`。

## 三、绑定以配置文件为准

绑定写在 `.github/custom-worker/wrangler-et.toml` 里，每次部署由 Actions 自动对齐。

- 本项目 `_worker.js` **只使用 KV**（代码内无任何 D1 调用），因此配置里只有 `[[kv_namespaces]]`。
- `cf-et` 使用 ID 为 `c8e94cc15b564dbfaae3c2b694e22bac` 的 KV namespace。
- 以后若新增 R2 / D1 / 队列等绑定，按同样方式写入该文件即可，例如：
  ```toml
  [[d1_databases]]
  binding = "DB"
  database_name = "my-db"
  database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  ```
  （`database_id` 留空时 wrangler 会走自动创建/自动补齐并把 id 写回配置文件）
- 绑定是"以配置为唯一真源"：在面板上手动加的绑定会在下次部署时被清掉。

## 四、注意事项

1. **Cloudflare 面板的 Git 构建必须关闭。** 面板侧的 Workers Builds / Git 集成只读**仓库根目录**的
   wrangler 配置；本仓库的个人配置已移到 `.github/custom-worker/`，若面板构建仍开着，它会用默认配置部署出一个名为
   默认 Worker，可能与个人部署的 Worker 冲突。请到面板 `设置 → 构建` 中关闭 Git 集成。
2. `cf-et` 配置设置了 `workers_dev = false` 和 `preview_urls = false`，不会启用默认的 `workers.dev` 域名或预览 URL；配置文件也没有添加 routes/custom domains。原 `web-surfing` Worker 的配置文件已删除，但这**不会删除 Cloudflare 上的 Worker**；其线上路由和域名状态由 Cloudflare 当前设置决定。
3. `cf-et` 的 KV `id` 为 `c8e94cc15b564dbfaae3c2b694e22bac`；部署时会按该配置更新 KV 绑定。
4. **环境变量仍在面板维护**（`UUID`、`PROXYIP`、`SUB`、`ADMIN`、`OFF_LOG` 等）。`keep_vars = true` 与 `--keep-vars` 保留已存在 `cf-et` 的变量；首次创建 `cf-et` 时仍需为它单独配置所需变量。
5. **手动部署 `cf-et` 时必须显式指定 `--config`**（根目录 `wrangler.toml` 保持上游配置，Wrangler 不会自动搜索 `.github/` 子目录）：
   ```bash
   npx wrangler deploy --config .github/custom-worker/wrangler-et.toml
   ```
   从根目录裸跑 `npx wrangler deploy` 会使用上游的根目录配置，而不是个人 Worker 的配置。
6. **wrangler 版本固定在 `.github/workflows/custom-deploy.yml`**（当前 `4.135.0`，要求 Node ≥ 22）。升级时改版本号即可。
7. 未启用 `--strict`：该参数会在检测到面板与配置存在差异时**直接拒绝部署**，而你目前在面板管理
   路由与环境变量，容易误阻断。如需强一致校验，可在部署命令中加上 `--strict`。
