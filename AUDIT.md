# Proxy Lite Audit

审计日期：2026-06-15

审计范围：轻量 A/B 中转落地、AnyTLS、NaiveProxy、协议级出站选择、A-B 内部 Shadowsocks、安全升级与失败回退、脚本交互、CI、公开文档脱敏。

> 公开仓库不得包含真实服务器 IP、真实域名、登录端口、节点密码、B 上游凭据、订阅链接、token、证书/key 文件或本地凭据文件。

发布状态：独立 GitHub 仓库 `https://github.com/jiasongji/proxy-lite` 已创建；本轮目标版本为 `v0.1.1`，Release assets 为 `proxy-lite.sh` 与 `proxy-lite.sh.sha256`；最终脚本 SHA-256 以 GitHub Release 附带校验文件为准。

## 1. 本轮整改目标

- [x] 保留独立命令：`PL` 与 `proxy-lite`，不占用完整项目的 `p-m`。
- [x] 保留 `entry_a` 与 `egress_b` A/B 拓扑。
- [x] `entry_a` 用户入口仅支持 AnyTLS / NaiveProxy。
- [x] 支持只部署 AnyTLS、只部署 NaiveProxy，或同时部署两者。
- [x] AnyTLS 与 NaiveProxy 可分别选择 `direct` 或 `egress-b`。
- [x] `entry_a` 不再提供 Shadowsocks 用户入口。
- [x] Shadowsocks 仅用于 A-B 内部链路：A 的 `egress-b` outbound 与 B 的 `ss-landing-in` inbound。
- [x] `entry_a` 不再依赖单个全局 B 出口；经 B 行为由 inbound tag route rule 明确控制。
- [x] 保留 `backup list`、`rollback`、`upgrade` 安全升级/回退能力。

## 2. 已裁剪功能

- [x] 无多用户数据库：不创建 `config/users.json`。
- [x] 无 `user` 命令。
- [x] 无域名/IP/AI 分流：无 `route` 命令、无 `route.rule_set`、无 AI/Google/custom 规则。
- [x] 无流量与限额：无 `stats`、`traffic`、`quota` 命令。
- [x] 无 V2Ray API stats / grpcurl / quota cron 依赖。
- [x] 无用户 Shadowsocks 客户端导出。

## 3. 静态检查

计划/本地检查项：

```bash
bash -n proxy-lite.sh
bash proxy-lite.sh help
bash proxy-lite.sh backup help
bash proxy-lite.sh rollback help
bash proxy-lite.sh upgrade help
```

如本机可用，执行：

```bash
shellcheck proxy-lite.sh
```

## 4. 生成配置烟测矩阵

临时目录 smoke 应覆盖：

- [x] `entry_a` only AnyTLS direct：只生成 `anytls-in`，不生成 `egress-b` outbound。
- [x] `entry_a` only NaiveProxy via B：生成 `naive-in` 与 `egress-b` outbound。
- [x] `entry_a` AnyTLS direct + NaiveProxy via B：AnyTLS 从 A 出口，NaiveProxy 从 B 出口。
- [x] `entry_a` AnyTLS via B + NaiveProxy direct：反向混合模式。
- [x] `entry_a` AnyTLS + NaiveProxy 全部 via B：兼容旧 A/B 落地行为，但通过 route rules 实现。
- [x] `entry_a` AnyTLS + NaiveProxy 全部 direct：不要求 B 参数，不生成 `egress-b` outbound。
- [x] `egress_b` 只生成 `ss-landing-in` Shadowsocks inbound 与 `direct` outbound。
- [x] `entry_a` 的 `route.final` 为 `direct`。
- [x] `entry_a` 不生成 `route.rule_set`。
- [x] 不生成 `config/users.json`。
- [x] 客户端导出目录不包含 B 上游 Shadowsocks 密码。
- [x] 使用上游 `ghcr.io/sagernet/sing-box:latest` 执行真实 `check -c`。

关键断言示例：

```bash
jq -e '.route.final == "direct" and (.route.rule_set == null)' "$tmp/config/sing-box.json"
jq -e '.route.rules[] | select((.inbound == ["anytls-in"]) and .action == "route" and .outbound == "direct")' "$tmp/config/sing-box.json"
jq -e '.route.rules[] | select((.inbound == ["naive-in"]) and .action == "route" and .outbound == "egress-b")' "$tmp/config/sing-box.json"
jq -e '[.inbounds[] | select(.tag=="ss-in")] | length == 0' "$tmp/config/sing-box.json"
! find "$tmp/config/client" -name '*shadowsocks*' -print -quit | grep -q .
! grep -R "<B_SS_PASSWORD>" "$tmp/config/client"
```

## 5. 对抗测试

- [x] `entry_a --components ss` 必须失败。
- [x] `entry_a --components anytls,ss` 必须失败。
- [x] `entry_a` 有协议选择 `egress-b` 但缺少 B 参数时必须失败。
- [x] 旧 env 中 `PM_NODE_ROLE=entry_a ENABLE_SS=1` 不得生成用户 SS 入站。
- [x] 旧 env 中 `SS_PASSWORD` 不得进入客户端导出。
- [x] `PL change-port ss` 在 `entry_a` 必须失败。
- [x] `PL change-secret ss` 在 `entry_a` 必须失败。
- [x] `egress_b` 上 `ss` 仅作为旧版 alias 兼容，推荐使用 `b-ss`。

## 6. 安全升级与回退

- [x] `PL upgrade --image ghcr.io/sagernet/sing-box:latest` 应拉取候选镜像、重新渲染配置并执行真实 `sing-box check -c`。
- [x] 候选镜像无法拉取时应返回非零并恢复更新前配置。
- [x] `PL backup list` 应列出包含 `lite.env`、`sing-box.json`、`docker-compose.yml` 和脚本副本的快照。
- [x] `PL rollback latest|TIMESTAMP` 应恢复配置并通过 `sing-box check -c`。

## 7. 已知限制

- Proxy Lite 是轻量项目，不提供多用户隔离、按用户导出、流量统计、限额或域名/IP/AI split 分流。
- 协议级出站只按入口 tag 控制，不按目标域名、IP、国家或 AI 服务分类。
- `PL upgrade` 依赖服务器能拉取目标 Docker 镜像；镜像仓库不可达时只能恢复配置，不能解决网络可达性问题。
- 回退快照覆盖关键运行配置，不替代完整系统级备份；证书文件、Docker daemon、云防火墙和外部镜像仓库仍需单独维护。
- 修改 A-B 内部 `b-ss` 密码或端口后，需要同步另一端配置。

## 8. 安全结论

- 公开文档仅使用占位主机、端口和密码。
- B 上游 Shadowsocks 凭据只保存在 A 的 `lite.env` 与服务端配置中，不写入用户客户端导出。
- `entry_a` 不开放用户 Shadowsocks 入站，不导出 Shadowsocks 客户端。
- `PL` 与 `proxy-lite` 不占用完整项目的 `p-m` / `proxy-manager` 命令。
- 生产运维命令要求 root 用户执行，避免权限不一致导致部署或回退失败。
