# Warden Worker 学习笔记

> 一个把 Bitwarden 兼容服务跑在 Cloudflare Workers 上的开源项目。我 fork 后用 GitHub Actions 部署到自己的 Cloudflare 账号，并读了它的实现。
> 代码来自上游 [qaz741wsd856/warden-worker](https://github.com/qaz741wsd856/warden-worker)，本仓库没有改动代码；上游说明见 [README.md](README.md)（保持原样，方便与上游同步）。

## 问题

自托管密码管理（如 Vaultwarden）需要一台长期在线的服务器，要维护、要续费，忘了续费就可能丢掉密码库。
这个项目想用 Cloudflare Workers 加 D1 数据库，做一个免费、免运维、兼容官方 Bitwarden 客户端的个人密码库。

## 方案（上游实现，学习要点）

- **运行时**：Rust 编译到 `wasm32-unknown-unknown`，由 `worker-build` 打包成 `build/index.js`，跑在 Cloudflare Workers 上（[wrangler.toml](wrangler.toml)）。
- **API**：[src/router.rs](src/router.rs) 只实现官方客户端用得到的子集：`/identity/connect/token` 登录换令牌、`/api/sync` 全量同步、密码条目（cipher）的增删改、软删除、恢复和导入。共享、Send、组织等高级功能不支持。
- **存储**：D1（托管 SQLite），只有 `users`、`ciphers`、`folders` 三张表（[sql/schema.sql](sql/schema.sql)），表结构变更走 `migrations/`。
- **密码学**：客户端只上传主密码的哈希；服务端再用 WebCrypto 的 PBKDF2-SHA256 加盐派生一次再存（`hash_password_for_storage`），校验时做恒定时间比较（`verify_password`），见 [src/crypto.rs](src/crypto.rs)。
- **部署流水线**：[push-cloudflare.yaml](.github/workflows/push-cloudflare.yaml) 安装 Rust 和 wasm 目标，用 secret 替换 `wrangler.toml` 里的 `D1_DATABASE_ID`，带重试地执行 D1 迁移，再部署 Worker。

## 结果

- 2025-12-02 通过 Actions 部署到自己的 Cloudflare 账号：第一次构建失败，第二次构建成功（[成功的构建](https://github.com/cowdedroyal-lab/warden-worker/actions/runs/19848709388)），随后前端也部署成功（[Deploy Frontend](https://github.com/cowdedroyal-lab/warden-worker/actions/runs/19850723452)）。
- 没有量化指标。

## 复现

1. Fork 本仓库，在 Cloudflare 建一个 D1 数据库。
2. 在仓库 Settings → Secrets 里配置 `CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`、`D1_DATABASE_ID`。
3. 在 Actions 里手动运行 **Build**；详细步骤见上游 [README.md](README.md) 的 Deployment 一节。

注意：**推送到 `main` 会自动触发生产部署**。只改文档的提交请在提交信息里写 `[skip ci]`。

## 维护提醒

截至 2026-11-08，本 fork 落后上游 179 个提交。这是密码库服务，上游的修复值得同步；同步会触发一次重新部署。

## 证据清单

[.portfolio/project.yaml](.portfolio/project.yaml) 记录了做过的事和对应证据。校验：`profile-atlas check .`
