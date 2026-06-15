# Proxy Lite

Proxy Lite 是 `proxy-manager` 的独立轻量项目，面向只需要服务器 A/B 中转落地的场景。服务器 A 只提供 AnyTLS 与 NaiveProxy 用户入口；服务器 B 只提供 Shadowsocks landing。AnyTLS 与 NaiveProxy 可以分别选择从服务器 A 本机 `direct` 出口，或经服务器 B 的 Shadowsocks 内部链路 `egress-b` 出口。

默认命令：

```bash
PL
# 等同于
proxy-lite
```

> `PL` 是大写命令，Linux 命令大小写敏感。生产运维命令请在 root 用户下执行。Proxy Lite 不占用 `p-m`，可与完整项目 `proxy-manager` 共存。

## 发布信息

- GitHub：`https://github.com/jiasongji/proxy-lite`
- Latest Release：`v0.1.1`
- Release assets：`proxy-lite.sh`、`proxy-lite.sh.sha256`
- `v0.1.1` 脚本 SHA-256：`408db808c2b2116e4b3407cfa7740dd1bca4595047181a5a89c1c280b16167b6`

## 功能范围

保留：

- `entry_a`：服务器 A，提供 AnyTLS / NaiveProxy 用户入口。
- `egress_b`：服务器 B，提供 A-B 内部 Shadowsocks landing，并从 B direct 出口。
- 协议级出站控制：AnyTLS 和 NaiveProxy 可分别选择 `direct` 或 `egress-b`。
- 支持只部署 AnyTLS、只部署 NaiveProxy，或同时部署两者。
- Docker host network 部署。
- TLS 证书挂载与 `sing-box check -c` 配置校验。
- 安全升级与失败回退：`PL upgrade`、`PL backup list`、`PL rollback latest`。

已裁剪：

- 无 Shadowsocks 用户入口：Shadowsocks 只用于服务器 A-B 内部链路。
- 无多用户：不创建 `config/users.json`，无 `user` 命令。
- 无域名/IP/AI 分流：无 `route` 命令、无 `route.rule_set`、无 AI/Google/custom 规则。
- 无流量限制：无 `stats`、`traffic`、`quota` 命令。

## 快速安装（一键复制）

在目标服务器 root 用户下执行。GitHub 页面会为下面的代码块提供复制按钮，可直接复制粘贴：

```bash
curl -fsSL https://github.com/jiasongji/proxy-lite/releases/latest/download/proxy-lite.sh -o /tmp/proxy-lite.sh
bash /tmp/proxy-lite.sh install
```

测试 main 分支开发版：

```bash
curl -fsSL https://raw.githubusercontent.com/jiasongji/proxy-lite/main/proxy-lite.sh -o /tmp/proxy-lite.sh
bash /tmp/proxy-lite.sh install
```

## A/B 部署示例

### 1. 服务器 B：Shadowsocks landing（一键复制）

把占位符替换为你的域名、服务器显示地址、端口和 A-B 内部链路密码后，在服务器 B 的 root 用户下执行：

```bash
PL install --yes \
  --node-role egress_b \
  --domain b.example.com \
  --server-ip 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>'
```

建议在服务器 B 的云安全组、UFW/firewalld 或面板防火墙中，仅允许服务器 A 的公网 IP 访问 B 的 Shadowsocks 端口。

### 2. 服务器 A：AnyTLS direct（一键复制）

只部署 AnyTLS，并让 AnyTLS 直接从服务器 A 出口，不需要填写服务器 B 参数：

```bash
PL install --yes \
  --node-role entry_a \
  --domain a.example.com \
  --server-ip 203.0.113.10 \
  --components anytls \
  --anytls-egress direct \
  --cert-file /path/fullchain.pem \
  --key-file /path/privkey.pem \
  --anytls-port 30001
```

### 3. 服务器 A：NaiveProxy 经 B 落地（一键复制）

只部署 NaiveProxy，并让 NaiveProxy 经服务器 B 的 Shadowsocks 内部链路出口：

```bash
PL install --yes \
  --node-role entry_a \
  --domain a.example.com \
  --server-ip 203.0.113.10 \
  --components naive \
  --naive-egress egress-b \
  --cert-file /path/fullchain.pem \
  --key-file /path/privkey.pem \
  --naive-port 30002 \
  --b-ss-host 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>'
```

### 4. 服务器 A：AnyTLS direct + NaiveProxy 经 B（一键复制）

这是常见混合模式：用户都连接服务器 A，但 AnyTLS 从 A 出口，NaiveProxy 经 B 出口。

```bash
PL install --yes \
  --node-role entry_a \
  --domain a.example.com \
  --server-ip 203.0.113.10 \
  --components anytls,naive \
  --anytls-egress direct \
  --naive-egress egress-b \
  --cert-file /path/fullchain.pem \
  --key-file /path/privkey.pem \
  --anytls-port 30001 \
  --naive-port 30002 \
  --b-ss-host 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>'
```

### 5. 其他组合

- AnyTLS 经 B、NaiveProxy direct：`--anytls-egress egress-b --naive-egress direct`
- AnyTLS 与 NaiveProxy 都经 B：`--anytls-egress egress-b --naive-egress egress-b`
- AnyTLS 与 NaiveProxy 都 direct：`--anytls-egress direct --naive-egress direct`，且不需要 B 参数

## 生成配置行为

`entry_a` 不做域名/IP/AI split 分流，只按用户入口协议 tag 路由：

```json
{
  "route": {
    "rules": [
      { "inbound": ["anytls-in"], "action": "route", "outbound": "direct" },
      { "inbound": ["naive-in"], "action": "route", "outbound": "egress-b" }
    ],
    "final": "direct"
  }
}
```

说明：

- `direct` 表示服务器 A 本机出口。
- `egress-b` 表示服务器 A 通过 Shadowsocks 连接服务器 B，再由 B direct 出口。
- 只有当至少一个协议选择 `egress-b` 时，A 才会生成 `egress-b` outbound 并要求填写 `B_SS_*` 参数。
- `route.final` 保持 `direct`，经 B 行为必须由明确的 inbound tag rule 控制。
- 不生成 `route.rule_set`。

## 目录结构

```text
/www/wwwroot/<domain>/Proxy-Lite/
├── bin/proxy-lite.sh
├── config/lite.env
├── config/sing-box.json
├── config/client/
├── compose/docker-compose.yml
├── logs/
├── backup/
├── runtime/
└── docs/Proxy-Lite-部署运维手册.html
```

客户端配置生成到 `config/client/`：

- `anytls-outbound.json`（启用 AnyTLS 时）
- `naive-outbound.json`（启用 NaiveProxy 时）
- `full-test-client.json`

`B_SS_PASSWORD` 是 A-B 内部链路密码，不会写入用户客户端导出。服务器 B 不生成客户端导出。

## 常用命令

```bash
PL install          # 安装 / 重新部署 / 选择 A 或 B 角色
PL update           # 从 GitHub 更新 Proxy Lite 脚本
PL upgrade          # 安全升级 sing-box 镜像：拉取、校验、失败回退
PL pull-image       # 仅拉取当前配置中的 Docker 镜像
PL backup list      # 列出可回退配置快照
PL rollback latest  # 回退到最新配置快照
PL env-check        # 检查 Docker、Compose、jq 和命令映射
PL start            # 启动服务
PL stop             # 停止服务
PL restart          # 重启服务
PL status           # 查看容器、端口和最近日志
PL logs             # 查看实时日志
PL info             # 查看节点与客户端配置路径
PL check            # 检查 sing-box 配置
PL doctor           # 运行诊断
PL topology         # 查看当前拓扑
PL change-port anytls 30001
PL change-port naive 30002
PL change-port b-ss 30003
PL change-secret anytls
PL change-secret naive
PL change-secret b-ss
PL uninstall        # 卸载清理
```

> `PL change-secret b-ss` 会修改 A-B 内部链路密码。若在 A 上修改，需要同步更新 B；若在 B 上修改，需要同步更新 A。

## 安全升级与回退

```bash
PL update
PL backup list
PL upgrade --image ghcr.io/sagernet/sing-box:latest
PL check
PL status
```

`PL upgrade` 会先备份当前配置，拉取候选镜像，重新渲染配置，并使用候选镜像执行真实 `sing-box check -c`。校验通过才会应用；候选镜像拉取、配置渲染或配置检查失败时，会恢复更新前快照。

手动回退：

```bash
PL backup list
PL rollback latest
PL rollback 20260615-120000
```

## 审计要求

发布前至少完成：

- `bash -n proxy-lite.sh`
- 如可用，执行 `shellcheck proxy-lite.sh`
- CLI smoke：`help`、`backup help`、`rollback help`、`upgrade help`
- 生成配置矩阵：AnyTLS only、NaiveProxy only、双协议 mixed、all direct、all via B、B landing
- 对抗测试：拒绝 `entry_a --components ss`，旧 `ENABLE_SS=1` 不生成用户 SS 入站，经 B 缺少 B 参数时失败
- Docker `sing-box check -c` 真实检查
- 安全升级 smoke：有效镜像通过、无效镜像失败并恢复快照
- 脱敏审计：公开文件不包含真实服务器 IP、域名、SSH 端口、节点密码、B 上游凭据、token、私钥、证书/key 路径或订阅链接

详见 `AUDIT.md`。
