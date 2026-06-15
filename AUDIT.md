# Proxy Lite Audit

审计日期：2026-06-16

审计范围：三协议（AnyTLS / NaiveProxy / Shadowsocks）平级用户入口、服务器 A/B 均可作入口、协议级出站选择、Shadowsocks 用户入口与 A-B 内部落地的凭据隔离、安全升级与失败回退、脚本交互、CI、公开文档脱敏。

> 公开仓库不得包含真实服务器 IP、真实域名、登录端口、节点密码、B 上游凭据、订阅链接、token、证书/key 文件或本地凭据文件。

发布状态：独立 GitHub 仓库 `https://github.com/jiasongji/proxy-lite`；本轮发布 `v0.1.2`，Release assets 为 `proxy-lite.sh` 与 `proxy-lite.sh.sha256`，`latest/download/proxy-lite.sh` 下载校验通过（SHA-256 见 README，发布后回填）。

## 1. 本轮整改目标（v0.1.2，相对 v0.1.1）

- [x] AnyTLS / NaiveProxy / Shadowsocks 三协议**平级**，可作为用户入口；`--components` 接受 `ss`，`all` 等价于 `anytls,naive,ss`。
- [x] 服务器 A（`entry_a`）与服务器 B（`egress_b`）**都可直接提供**用户入口（任一/组合/全部）。
- [x] 每个启用的入口协议（含 SS）可独立选择 `direct` 或 `egress-b`；`egress_b` 的入口恒直出。
- [x] Shadowsocks **同时**保留 A→B 内部落地角色：A 的 `egress-b` 出站、B 的 `ss-landing-in` 入站。
- [x] 用户 SS 入口（`ss-in`）与 A-B 落地（`ss-landing-in`）是两套独立入站：端口、密码互不相同；落地凭据不进入用户客户端导出。
- [x] `route.final` 恒为 `direct`；经 B 行为由 inbound tag 规则明确控制；不生成 `route.rule_set`。
- [x] v0.1.1 安全迁移：旧 env 不含 `ENABLE_SS_ENTRY` 时按「未启用 SS 用户入口」处理，行为与 v0.1.1 一致。

## 2. 已裁剪功能（与完整版 proxy-manager 的区别）

- [x] 无多用户数据库：不创建 `config/users.json`。
- [x] 无 `user` 命令。
- [x] 无域名/IP/AI 分流：无 `route` 命令、无 `route.rule_set`、无 AI/Google/custom 规则。
- [x] 无流量与限额：无 `stats`、`traffic`、`quota` 命令。
- [x] 无 V2Ray API stats / grpcurl / quota cron 依赖。

## 3. 变量模型

| 变量 | 含义 |
|---|---|
| `ENABLE_ANYTLS` / `ENABLE_NAIVE` | AnyTLS / NaiveProxy **用户入口**（两角色均可） |
| `ENABLE_SS` | **内部 SS 落地**（`egress_b`=1、`entry_a`=0；语义同 v0.1.1） |
| `ENABLE_SS_ENTRY` | **用户 SS 入口**（两角色均可，默认 0） |
| `SS_PORT` / `SS_METHOD` / `SS_PASSWORD` | **用户 SS 入口** 监听端口/method/密码 |
| `B_SS_HOST/PORT/METHOD/PASSWORD` | A→B 落地凭据（entry_a=远端目标；egress_b=B 本机监听；不导出） |
| `ANYTLS_EGRESS` / `NAIVE_EGRESS` / `SS_EGRESS` | 各入口协议出站（entry_a: direct\|egress-b；egress_b: 强制 direct） |

迁移不变量：`ENABLE_SS` 保留 v0.1.1 语义（落地），新增 `ENABLE_SS_ENTRY` 控制用户入口；旧 env 缺省 `ENABLE_SS_ENTRY` ⇒ 不生成 `ss-in`、不导出 SS 客户端 ⇒ 与 v0.1.1 行为一致。

## 4. 静态检查

- [x] `bash -n proxy-lite.sh` 通过。
- [x] `shellcheck proxy-lite.sh` 通过（CI 与本机 Docker `koalaman/shellcheck:stable`）。
- [x] CLI help smoke：`help`、`backup help`、`rollback help`、`upgrade help`。
- [x] 公开文档中无 GitHub/Docker 未替换占位。
- [x] README/AUDIT/HTML 未命中私钥、token、AWS key、测试用密码、旧本地凭据文件名。

## 5. 生成配置烟测矩阵

本地与 CI 临时目录 smoke 覆盖：

- [x] `entry_a` anytls direct / naive via B（单协议）。
- [x] `entry_a` ss-only direct：仅 `ss-in`、有 `shadowsocks-outbound.json`、无 `egress-b`。
- [x] `entry_a` ss via B：`ss-in -> egress-b`、有 `egress-b` 出站。
- [x] `entry_a` anytls+naive+ss 三入口混合（anytls direct / naive egress-b / ss direct）。
- [x] `entry_a` 全 via B（三协议均 egress-b）/ 全 direct（无 B 参数、无 `egress-b`）。
- [x] `egress_b` 仅落地：`ss-landing-in` + `direct`，无任何用户入口，无客户端导出。
- [x] `egress_b` 落地 + anytls 入口：同时有 `ss-landing-in` 与 `anytls-in`，导出含 `anytls-outbound.json`，不含落地密码。
- [x] `egress_b` 落地 + ss 入口：同时有 `ss-landing-in`（`B_SS_*`）与 `ss-in`（`SS_*`），两端口/密码不同；导出 `shadowsocks-outbound.json` 使用**用户** SS 密码，不含 `B_SS_PASSWORD`。
- [x] `entry_a` 的 `route.final` 恒为 `direct`；不生成 `route.rule_set`。
- [x] 使用上游 `ghcr.io/sagernet/sing-box:latest` 对 ss-only、三入口混合、egress_b 落地+ss 入口等代表配置执行真实 `check -c`。

关键断言示例：

```bash
jq -e '.route.final == "direct" and (.route.rule_set == null)' "$tmp/config/sing-box.json"
jq -e '.route.rules[] | select((.inbound == ["ss-in"]) and .outbound == "egress-b")' "$tmp/config/sing-box.json"
jq -e '.inbounds[] | select(.tag=="ss-landing-in" and .password == "<LANDING>")' "$tmp/config/sing-box.json"
jq -e '.inbounds[] | select(.tag=="ss-in" and .password == "<USER>")' "$tmp/config/sing-box.json"
! grep -R "<LANDING>" "$tmp/config/client"
```

## 6. 对抗测试

- [x] `entry_a` ss via B 缺少 `B_SS_*` 参数时失败。
- [x] `egress_b` 用户 SS 入口端口与落地端口相同时失败（端口重复）。
- [x] 落地 `B_SS_PASSWORD` 不进入客户端导出（即便节点同时有用户 SS 入口）。
- [x] 旧 v0.1.1 entry_a env（`ENABLE_SS=1` 无 `ENABLE_SS_ENTRY`）不生成用户 SS 入站。
- [x] 旧 v0.1.1 egress_b env（`ENABLE_SS=1` 落地）仅生成落地、无用户 SS 入站。
- [x] 未启用 SS 入口时 `change-port ss` / `change-secret ss` 失败并提示「未启用 Shadowsocks 用户入口」；启用时成功。
- [x] 非法 `--node-role`、未知 `--components`、缺 env 等均按预期失败。

## 7. 安全升级与回退

- [x] `PL upgrade --image ghcr.io/sagernet/sing-box:latest` 拉取候选镜像、重新渲染配置并执行真实 `sing-box check -c`。
- [x] 候选镜像无法拉取时返回非零并恢复更新前配置。
- [x] `PL backup list` 列出含 `lite.env`、`sing-box.json`、`docker-compose.yml` 与脚本副本的快照。
- [x] `PL rollback latest|TIMESTAMP` 恢复配置并通过 `sing-box check -c`。

## 8. 已知限制

- Proxy Lite 是轻量项目，不提供多用户隔离、按用户导出、流量统计、限额或域名/IP/AI split 分流。
- 协议级出站只按入口 tag 控制，不按目标域名、IP、国家或 AI 服务分类。
- `egress_b` 始终提供 `ss-landing-in`（落地角色）；如不需要 A 经 B，可仅使用 B 的用户入口，落地入站空转无害。
- B 上的「用户 SS 入口」与「A-B 落地」是两套独立入站，必须使用不同端口与密码。
- 修改 A-B 内部 `b-ss` 密码或端口后，需要同步另一端配置。
- `PL upgrade` 依赖服务器能拉取目标 Docker 镜像；镜像仓库不可达时只能恢复配置，不能解决网络可达性。

## 9. 安全结论

- 公开文档仅使用占位主机、端口和密码。
- A-B 落地 Shadowsocks 凭据只保存在服务端 `lite.env` 与 sing-box 配置中，不写入用户客户端导出；与用户 SS 入口凭据物理隔离（不同端口/密码）。
- `route.final` 恒为 `direct`，经 B 行为由 inbound tag 规则明确控制。
- `PL` 与 `proxy-lite` 不占用完整项目的 `p-m` / `proxy-manager` 命令。
- 生产运维命令要求 root 用户执行，避免权限不一致导致部署或回退失败。
