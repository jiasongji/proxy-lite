# Proxy Lite

Proxy Lite 是 `proxy-manager` 的独立轻量项目，面向只需要服务器 A/B 中转落地的场景。**AnyTLS / NaiveProxy / Shadowsocks 三种协议平级**：服务器 A 与服务器 B 都可以直接对用户提供其中任意一种、几种或全部作为用户入口；每个启用的入口协议都可以独立选择从本机 `direct` 出口，或经对端服务器 B 的 Shadowsocks 内部链路 `egress-b` 落地。Shadowsocks 同时承担服务器 A→B 的内部落地链路（A 的 `egress-b` 出站、B 的 `ss-landing-in` 入站），落地凭据不进入用户客户端导出。

默认命令：

```bash
PL
# 等同于
proxy-lite
```

> `PL` 是大写命令，Linux 命令大小写敏感。生产运维命令请在 root 用户下执行。Proxy Lite 不占用 `p-m`，可与完整项目 `proxy-manager` 共存。

## 发布信息

- GitHub：`https://github.com/jiasongji/proxy-lite`
- Latest Release：`v0.1.2`
- Release assets：`proxy-lite.sh`、`proxy-lite.sh.sha256`
- `v0.1.2` 脚本 SHA-256：`3d3372bc23f9ab03ceed3501f0d6f279a4b830f8243031b00ff796eb7abdec5d`

## 功能范围

保留：

- `entry_a`：服务器 A，提供 AnyTLS / NaiveProxy / Shadowsocks 用户入口（任选）。
- `egress_b`：服务器 B，可直接提供 AnyTLS / NaiveProxy / Shadowsocks 用户入口，并同时提供 A-B 内部 Shadowsocks 落地。
- 协议级出站控制：每个启用的入口协议可分别选择 `direct` 或 `egress-b`。
- 支持只部署任意一种协议、任意组合，或全部部署；A、B 两台服务器均可作为入口节点。
- Docker host network 部署。
- TLS 证书挂载与 `sing-box check -c` 配置校验。
- 安全升级与失败回退：`PL upgrade`、`PL backup list`、`PL rollback latest`。

已裁剪（与完整版 `proxy-manager` 的区别）：

- 无多用户：不创建 `config/users.json`，无 `user` 命令。
- 无域名/IP/AI 分流：无 `route` 命令、无 `route.rule_set`、无 AI/Google/custom 规则。
- 无流量限制：无 `stats`、`traffic`、`quota` 命令。

## 概念：用户入口 vs 内部落地

| 概念 | 含义 | 配置 / 凭据 | 是否导出给用户 |
|---|---|---|---|
| 用户入口（anytls / naive / ss） | 终端用户直接连接的入站协议 | `--components` 选择；`*_PORT`、`*_PASSWORD` | 是 |
| A-B 内部落地（b-ss） | A 经 B 出口时使用的 Shadowsocks 链路 | `--b-ss-*`；A 端为 `egress-b` 出站，B 端为 `ss-landing-in` 入站 | 否 |

> 同一台服务器上「用户 SS 入口」与「A-B 落地」是两套独立的 Shadowsocks 入站：端口、密码互不相同。用户入口凭据会写入 `config/client/`；落地凭据只保存在服务端，绝不外泄。

## 快速安装（一键复制）

在目标服务器 root 用户下执行：

```bash
curl -fsSL https://github.com/jiasongji/proxy-lite/releases/latest/download/proxy-lite.sh -o /tmp/proxy-lite.sh
bash /tmp/proxy-lite.sh install
```

测试 main 分支开发版：

```bash
curl -fsSL https://raw.githubusercontent.com/jiasongji/proxy-lite/main/proxy-lite.sh -o /tmp/proxy-lite.sh
bash /tmp/proxy-lite.sh install
```

## 部署示例

> 占位符请替换为你的域名、服务器地址、端口与密码。A-B 落地密码（`<B_SS_PASSWORD>`）与用户入口密码（如 `<USER_SS_PASSWORD>`）应使用不同的值。

### 1. 服务器 B：仅作 A-B Shadowsocks 落地（一键复制）

```bash
PL install --yes \
  --node-role egress_b \
  --domain b.example.com \
  --server-ip 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>'
```

建议在服务器 B 的云安全组 / UFW / 面板防火墙中，仅允许服务器 A 的公网 IP 访问 B 的 Shadowsocks 落地端口。

### 2. 服务器 B：落地 + 直接提供 AnyTLS 入口（一键复制）

B 既作为 A 的落地，又直接对用户提供 AnyTLS 入口（用户连 B，从 B 直出）：

```bash
PL install --yes \
  --node-role egress_b \
  --domain b.example.com \
  --server-ip 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>' \
  --components anytls \
  --cert-file /path/fullchain.pem \
  --key-file /path/privkey.pem \
  --anytls-port 30011
```

### 3. 服务器 B：落地 + 直接提供 Shadowsocks 入口（一键复制）

B 同时跑「内部落地」与「用户 SS 入口」两个独立 Shadowsocks 入站：

```bash
PL install --yes \
  --node-role egress_b \
  --domain b.example.com \
  --server-ip 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>' \
  --components ss \
  --ss-port 30014 \
  --ss-method aes-128-gcm \
  --ss-password '<USER_SS_PASSWORD>'
```

### 4. 服务器 A：只部署 Shadowsocks 入口，直出（一键复制）

```bash
PL install --yes \
  --node-role entry_a \
  --domain a.example.com \
  --server-ip 203.0.113.10 \
  --components ss \
  --ss-egress direct \
  --ss-port 30004 \
  --ss-password '<USER_SS_PASSWORD>'
```

### 5. 服务器 A：Shadowsocks 入口经 B 落地（一键复制）

```bash
PL install --yes \
  --node-role entry_a \
  --domain a.example.com \
  --server-ip 203.0.113.10 \
  --components ss \
  --ss-egress egress-b \
  --ss-port 30004 \
  --ss-password '<USER_SS_PASSWORD>' \
  --b-ss-host 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>'
```

### 6. 服务器 A：三协议混合出站（一键复制）

AnyTLS 直出、NaiveProxy 经 B、Shadowsocks 直出：

```bash
PL install --yes \
  --node-role entry_a \
  --domain a.example.com \
  --server-ip 203.0.113.10 \
  --components anytls,naive,ss \
  --anytls-egress direct \
  --naive-egress egress-b \
  --ss-egress direct \
  --cert-file /path/fullchain.pem \
  --key-file /path/privkey.pem \
  --anytls-port 30001 \
  --naive-port 30002 \
  --ss-port 30004 \
  --ss-password '<USER_SS_PASSWORD>' \
  --b-ss-host 198.51.100.20 \
  --b-ss-port 30003 \
  --b-ss-method aes-128-gcm \
  --b-ss-password '<B_SS_PASSWORD>'
```

### 7. 其他组合

- 只部署 AnyTLS / 只部署 NaiveProxy：`--components anytls` / `--components naive`。
- 全部经 B：所有 `--*-egress` 都用 `egress-b`，并填写 `--b-ss-*`。
- 全部直出：所有 `--*-egress` 都用 `direct`，无需 B 参数。
- AnyTLS 经 B、NaiveProxy 直出等任意组合：分别指定 `--anytls-egress` / `--naive-egress` / `--ss-egress` 即可。

## 生成配置行为

`entry_a` 不做域名/IP/AI split 分流，只按用户入口协议 tag 路由，`route.final` 恒为 `direct`：

```json
{
  "route": {
    "rules": [
      { "inbound": ["anytls-in"], "action": "route", "outbound": "direct" },
      { "inbound": ["naive-in"], "action": "route", "outbound": "egress-b" },
      { "inbound": ["ss-in"], "action": "route", "outbound": "direct" }
    ],
    "final": "direct"
  }
}
```

说明：

- `direct` 表示本机出口；`egress-b` 表示经对端 Shadowsocks 落地后再直出。
- 只有当至少一个入口协议选择 `egress-b` 时，`entry_a` 才会生成 `egress-b` 出站并要求填写 `--b-ss-*`。
- `egress_b` 的所有入口协议与落地流量都从本机 `direct` 出口，路由为 `{ "final": "direct" }`。
- 经 B 行为始终由明确的 inbound tag 规则控制，`route.final` 永远是 `direct`。
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

客户端配置生成到 `config/client/`（A、B 任一节点只要启用了用户入口就会导出）：

- `anytls-outbound.json`（启用 AnyTLS 入口时）
- `naive-outbound.json`（启用 NaiveProxy 入口时）
- `shadowsocks-outbound.json`（启用 Shadowsocks 入口时）
- `full-test-client.json`（至少启用一个入口时）

`B_SS_PASSWORD`（A-B 落地密码）不会写入用户客户端导出。仅作落地、未启用任何用户入口的服务器不生成客户端导出。

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
PL change-port ss 30004
PL change-port b-ss 30003
PL change-secret anytls
PL change-secret naive
PL change-secret ss
PL change-secret b-ss
PL uninstall        # 卸载清理
```

> `anytls` / `naive` / `ss` 指用户入口端口或密码；`b-ss` 指 A-B 内部落地端口或密码，修改后需同步另一端。`change-secret all` / `regen` 只轮换本机用户入口，不自动改 `b-ss`，避免两端不一致。

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
PL rollback 20260616-120000
```

## 从 v0.1.1 升级

v0.1.2 重新引入 Shadowsocks 用户入口，并允许服务器 B 直接提供入口。升级是安全的：

- 旧的 `entry_a` 部署（AnyTLS/NaiveProxy）行为不变：仍无 SS 用户入口，除非你显式 `--components ...ss...`。
- 旧的 `egress_b` 部署仍只作落地：仍无用户入口，除非你显式启用 `--components`。
- 新增 `ENABLE_SS_ENTRY` / `SS_EGRESS` 变量；旧 env 不含这两个变量时按「未启用 SS 用户入口」处理。

升级脚本后如需启用 SS 用户入口或 B 入口，重新执行 `PL install` 并带上新的 `--components` / `--ss-egress` 等参数即可。

## 审计要求

发布前至少完成：

- `bash -n proxy-lite.sh`
- 如可用，执行 `shellcheck proxy-lite.sh`
- CLI smoke：`help`、`backup help`、`rollback help`、`upgrade help`
- 生成配置矩阵：anytls/naive/ss 单协议与组合、ss direct、ss via B、三入口混合、全 direct、全 via B、B 仅落地、B 落地+anytls、B 落地+ss（双 SS 入站隔离）
- 对抗测试：经 B 缺 B 参数失败、SS 入口端口与落地端口冲突失败、SS 用户密码不外泄、旧 env 迁移不产生 SS 用户入站
- Docker `sing-box check -c` 真实检查
- 安全升级 smoke：有效镜像通过、无效镜像失败并恢复快照
- 脱敏审计：公开文件不包含真实服务器 IP、域名、SSH 端口、节点密码、B 上游凭据、token、私钥、证书/key 路径或订阅链接

详见 `AUDIT.md`。
