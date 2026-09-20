# Custom Deploy（个人部署工作流）

本目录下的文件是**个人专属文件**，上游 `cmliu/edgetunnel` 不存在这些路径，因此
`git rebase` / 合并上游时不会产生任何冲突。

| 文件 | 作用 |
| --- | --- |
| `custom-worker/wrangler.toml` | Cloudflare Worker 部署配置（worker 名、入口、绑定、兼容日期） |
| `.github/workflows/custom-deploy.yml` | 手动触发部署的 GitHub Actions 工作流 |

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

流程顺序：安装 wrangler → 校验 Secret → 打印账号信息 → dry-run 编译校验 → 正式部署。

## 三、绑定以配置文件为准

绑定写在 `custom-worker/wrangler.toml` 里，每次部署由 Actions 自动对齐，不需要再去面板手动添加。

- 本项目 `_worker.js` **只使用 KV**（代码内无任何 D1 调用），因此配置里只有 `[[kv_namespaces]]`。
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
   wrangler 配置；本仓库的配置已移到 `custom-worker/`，若面板构建仍开着，它会用默认配置部署出一个名为
   `edgetunnel` 的 Worker，与 `web-surfing` 冲突。请到面板 `设置 → 构建` 中关闭 Git 集成。
2. **KV 的 `id` 必须与面板中该 Worker 的绑定一致**（`custom-worker/wrangler.toml` 中 `[[kv_namespaces]]`）。
   不一致会导致部署把面板上的绑定覆盖掉。
3. **环境变量仍在面板维护**（`UUID`、`PROXYIP`、`SUB`、`ADMIN`、`OFF_LOG` 等）。
   配置文件中的 `keep_vars = true` 与命令行的 `--keep-vars` 保证部署不会清空面板变量。
4. **本地部署必须显式指定 `--config`**（根目录的 `wrangler.toml` 已删除，且 wrangler 不会搜索子目录）：
   ```bash
   npx wrangler deploy --config custom-worker/wrangler.toml
   ```
   不要从根目录裸跑 `npx wrangler deploy`：找不到配置时 wrangler 的 autoconfig 可能提示生成一个根目录
   `wrangler.toml`，那会重新引入与上游冲突的文件。
5. **wrangler 版本固定在 `custom-deploy.yml`**（当前 `4.135.0`，要求 Node ≥ 22）。升级时改版本号即可。
6. 未启用 `--strict`：该参数会在检测到面板与配置存在差异时**直接拒绝部署**，而你目前在面板管理
   路由与环境变量，容易误阻断。如需强一致校验，可在部署命令中加上 `--strict`。
