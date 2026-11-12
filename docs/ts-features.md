# Tailscale 增强版（`Alpha-ts` 分支）

本仓库是 [MetaCubeX/mihomo](https://github.com/MetaCubeX/mihomo) 的 fork，用途只有一个：
让 mihomo 内置的 **userspace Tailscale 节点**能当路由器用 —— 发布子网路由、固定 UDP 端口、
把本机服务转发进 tailnet、以及让 TUN 里的 ping 真正走 Tailscale。

## 与主线的关系

- **基线**：始终是上游最新 release 的 commit（当前 `v1.19.34` / `0455d0d2`）。
- **额外内容**：上游待合并 PR 里的 4 个 Tailscale 提交（来源见下一节），外加一个 CI 工作流。
- 相对基线的代码差异：**12 个新增文件 + 11 个修改文件，+1828 / −29**。
  除 Tailscale 相关文件和 `docs/config.yaml` 注释外，其余代码与主线完全一致。

## 来源：这些功能是从哪同步的

**不是自己写的，全部来自上游还没合并的 PR**，我们只是把它重放到 release tag 上：

| 环节 | 位置 | 状态 |
|---|---|---|
| 需求 | MetaCubeX/mihomo issue [#2829](https://github.com/MetaCubeX/mihomo/issues/2829)（作者 [@jsntit](https://github.com/jsntit)，2026-05-24） | open |
| 实现 | PR [#3248](https://github.com/MetaCubeX/mihomo/pull/3248) by [@long2ice](https://github.com/long2ice)，base `Alpha` | **open，未合并** |
| 代码分支 | `long2ice/mihomo` → `feat/tailscale-advertise-routes`，HEAD `1d4aedd6`（2026-10-05） | 已停止更新 |
| 文档 | MetaCubeX/Meta-Docs PR [#222](https://github.com/MetaCubeX/Meta-Docs/pull/222) | open |

PR 共 5 个提交 / 22 个文件，其中 4 个是功能提交，逐个 cherry-pick 到 `v1.19.34` 上：

| # | 功能 | 原始提交（long2ice） | 本分支重放后 | 日期 |
|---|---|---|---|---|
| 1 | `advertise-routes` 发布子网路由 | `560f713c` | `86bc4d17` | 2026-09-26 |
| 2 | TUN ICMP echo 经 Tailscale 发送 | `241f5aa2` | `f34b10dd` | 2026-09-26 |
| 3 | `port` 固定 UDP 端口（默认 41641） | `f2284736` | `0d167128` | 2026-09-28 |
| 4 | `host-forward` 入站转发到 loopback | `1d4aedd6` | `5a700660` | 2026-10-05 |

第 4 项是 PR 讨论里 [@UnlastingR](https://github.com/UnlastingR) 提出的 FlClash / Android 场景，
作者追加提交实现的。

**没有同步进来的部分**：

- `e16badcc`（PR 第 5 个提交）是 `merge Alpha into Tailscale feature branch`，纯同步动作，不需要；
- 分支的基点 `63bd52ec`（wwqgtxx 改默认 IP stack / congestion-controller）本来就是上游提交，
  已经包含在 `v1.19.34` 里，不存在需要额外带的东西。

**为什么用 cherry-pick 而不是直接 merge 分支**：该分支基于 `63bd52ec`（2026-09-27），
比当前基线落后一个半月，并且自带 merge commit；直接 merge 会把无关历史一起带进来。
挑功能提交重放后历史是直线，`git diff v1.19.34..HEAD` 正好等于补丁集，
`Author` 仍是原作者，提交信息里的 `cherry picked from` 保留原始 SHA 可回溯。

**上游合并后就可以退役本分支**：一旦 PR #3248 合进 `Alpha` 并出现在 release 里，
删掉这 4 个提交直接跟随主线即可，不需要长期分叉。

## 功能对比

| 能力 | 主线（MetaCubeX/mihomo） | 本分支 |
|---|---|---|
| `tailscale` 出站代理、`accept-routes`、`exit-node` | ✅ | ✅（与主线相同） |
| `advertise-routes`：把本机可达网段发布进 tailnet | ❌ | ✅ |
| `port`：固定 UDP 监听端口 | ❌（随机） | ✅ 默认 41641 |
| `host-forward`：tailnet 入站转发到本机 loopback 服务 | ❌ | ✅ |
| TUN 内 ping 目标时经 Tailscale 真正发 ICMP echo | ❌ | ✅（`NETWORK,ICMP` 规则） |

四项功能的动机依次是：**能做 subnet router**、**NAT/防火墙可预期**、**不用给每个服务开
Tailscale 客户端**、**ping 能反映真实链路而不是只测 DERP**。

## 1. `advertise-routes`：发布子网路由

```yaml
proxies:
  - name: ts
    type: tailscale
    auth-key: tskey-auth-xxxx
    state-dir: ./tailscale
    advertise-routes:
      - 192.168.1.0/24
      - fd12:3456:789a::/64
```

行为细节：

- 路由要在 **Tailscale / Headscale 控制台批准**之后，其他节点才会真正使用。
- 设为 `[]` 会**清除**已发布的路由。
- 只要配置了这个字段（包括 `[]`），节点就会在运行时主动启动；仅做配置检查（`-t`）不会启动节点。
- 同时发布 `0.0.0.0/0` 和 `::/0` 等于把自己声明成 **exit node**，此时不能再配 `exit-node`。
- 进入这些网段的 TCP/UDP 由 userspace 网络栈**从本机转发**出去，
  沿用 `interface-name`、`routing-mark`、`dialer-proxy`。
- 校验：空前缀、非法 CIDR、重复前缀都会在配置解析阶段报错。

## 2. `port`：固定 UDP 监听端口

```yaml
proxies:
  - name: ts
    type: tailscale
    port: 41641   # 默认 41641；设为 0 则用随机端口
```

默认值与原生 Tailscale 客户端一致（41641）。好处是防火墙规则可以写死，
NAT 打洞行为也更稳定。**防火墙必须放行实际监听的端口**（设 0 时要看日志确认随机端口）。

## 3. `host-forward`：把本机服务转发进 tailnet

```yaml
proxies:
  - name: ts
    type: tailscale
    host-forward:
      enabled: true
      target: 127.0.0.1   # 只允许 loopback，也可以填 ::1
      tcp: true           # 默认 true
      udp: true           # 默认 true，独立于上面的出站 udp
```

例如节点 IP 是 `100.104.118.62`，那么访问 `100.104.118.62:6767` 会被转发到本机
`127.0.0.1:6767`。路由器上的 LuCI、nginx、RPC 之类不用单独装客户端就能进 tailnet，
访问仍受 Tailscale ACL / grants 控制。

需要注意：

- 只在 `enabled: true` 时生效；`target` 只接受 loopback IP，其他值直接报配置错误。
- 服务看到的来源是**本地转发连接**，与原生客户端的系统 TUN 模式不同（不是原始 tailnet 源 IP）。
- 本机转发直连 loopback，**不经过** `dialer-proxy`、`interface-name`、`routing-mark`、`ip-version`。
- 启用后节点会在 provider 初始化/更新时主动启动（`StartBackground`），不需要先有出站流量才上线。

## 4. TUN ICMP echo 经 Tailscale

TUN 开启时，`ping` 一个命中 tailscale 出站的目标，旧行为只能测到 DERP 连通性；
现在会由 Tailscale 真正发 echo 探测，并只把目标的 ICMP 回复写回 TUN，
本节点自己的地址在上线后由本地回应。

规则层面新增了 `ICMP` 网络类型，可以显式分流：

```yaml
rules:
  - NETWORK,ICMP,DIRECT        # 让 ping 直接出去，不进 tailscale
  - DOMAIN,example.com,ts      # TCP/UDP 仍走 tailscale
```

行为细节：

- 按规则选中的代理如果不支持 ICMP（`SupportICMP() == false`），保留原来的直连行为。
- 规则命中 `REJECT` 时，请求被丢弃。
- 这是 **echo 代理**：探测包由 Tailscale 生成，**不支持** traceroute、MTU 探测或任意 ICMP 报文透传。

## 构建

功能代码的构建标签是 `with_gvisor && !no_tailscale`：

- `-tags with_gvisor`：启用 Tailscale（本仓库发布的产物都带这个标签）
- 加 `no_tailscale`：`tailscale` 出站直接报错「tailscale support is disabled」，
  与主线行为一致；配置里的新字段会被解析但不生效

CI 工作流 `.github/workflows/build-ts.yml` 一次性产出 36 个目标
（linux 19 / windows 5 / darwin 4 / freebsd 4 / android 4），产物命名与上游一致：

```
mihomo-<goos>-<output>-<版本>.gz    # windows 为 .zip
```

产物版本号自带 `-ts` 后缀，例如 `v1.19.34-ts`，`mihomo -v` 会显示这个版本。

## 同步上游

上游发新版后：

0. 先看 PR [#3248](https://github.com/MetaCubeX/mihomo/pull/3248) 是否已经合并 ——
   合并了就别再打 4 个补丁，直接跟主线；
1. 在临时 worktree 里把 4 个提交 cherry-pick 到新 tag 上，先确认会不会冲突；
2. 无冲突就在 `sync-<版本>` 分支上重放，然后 **merge 回 `Alpha-ts`**（不要 rebase / force-push，
   否则基线到分支的提交范围会乱）；
3. 验证编译与 `-v` 输出；
4. 触发 `Build (ts)` 工作流重新出包。

历史已知冲突面：`adapter/outbound/tailscale.go`（上游本来就有 tailscale 出站，改动最集中），
其次是 `tailscale_stub.go`、`base.go`、`constant/tunnel.go`、`rules/common/network_type.go`、
`tunnel/tunnel.go`；`docs/config.yaml` 基本手动保留两边即可。

## 已知限制

- 上游主线本身也在演进 tailscale 出站，未来合并可能需要语义重解，不能只看 diff。
- `advertise-routes` 的转发是 userspace 实现，性能不如内核路由，适合家庭 / 小型网络。
- `host-forward` 只支持单个 loopback 目标，不支持按端口映射到不同后端。
- ICMP 只做 echo，不能当 ping 隧道用。
