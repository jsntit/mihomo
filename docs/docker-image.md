# Docker 镜像（GHCR）

仓库自带上游的 `Dockerfile` + `docker/file-name.sh`，原本只给上游发布流水线用（推到 Docker Hub）。
`.github/workflows/docker.yml` 复用了这两个文件，把镜像发到你自己的 GitHub Container Registry。

## 用法

三种触发方式：

1. **手动**：Actions → **Docker (GHCR)** → Run workflow，按下表填参数。
2. **发版自动**：push 一个 `v*-ts` 的 tag，就会跟着 `Build (ts)` 一起出镜像 ——
   此时 `ref` 取该 tag、`version` 直接取 tag 名（不走 `git describe`，见下面坑）、`tag_latest=true`。
3. **定时巡检**：每周一 02:00 UTC（北京 10:00）自动检查，满足**任一**条件就出镜像：
   - 上游 `MetaCubeX/mihomo` 发了比 `BASELINE` 新的 release；
   - 本分支相对「上次出镜像的那个 commit」有新提交。

   两个条件都不满足就整段跳过，只留一条 Job Summary，不跑编译。分支自动构建**不抢 `latest`**。

| 参数 | 默认 | 说明 |
|---|---|---|
| `ref` | `Alpha-ts` | 打包哪个分支/tag。`Alpha-ts` = 带 Tailscale 增强；`Alpha` = 纯净主线 |
| `version` | 空 | 写进内核 `Version` 与镜像 tag；留空则按「tag 名 → 分支 describe → 短 sha」推断 |
| `platforms` | `linux/amd64,linux/arm64,linux/arm/v7,linux/386` | 可加 `linux/riscv64`。只用 `docker/file-name.sh` 支持的这五个 |
| `tag_latest` | `true` | 额外打 `latest`。**两套镜像同时维护时，只给其中一套开** |
| `push` | `true` | 关掉则只构建不推送，用于验证 |

### `BASELINE` 文件

仓库根目录的 `BASELINE` 记录「ts 分支当前跟上到哪个上游版本」，巡检拿它跟上游最新 release 比。
**把 ts 补丁重放到新上游之后，记得同步改这个文件**（`v1.19.34` 这种格式），否则巡检会反复判定
「上游有新版本」，每周都白构建一次。

> 巡检只负责「上游发版 → 出新镜像」，**不会**替你把补丁 rebase 到新上游。带 `-ts` 后缀的镜像
> 意味着该分支已合过那个版本；若分支还没跟上而巡检先跑了，镜像版本号会显得比实际新。
> 正确顺序是：按 skill `mihomo-ts-upstream-sync` 重放补丁 → 改 `BASELINE` → 推送。

### 巡检的节流依据

判断「这个 commit 有没有发过镜像」不看 tag（`latest` 由多个版本共享，`v1.19.34` 与
`v1.19.34-ts` 谁后跑谁占，靠 tag 反查会误判），而是读镜像 config 里的
`org.opencontainers.image.revision`（`metadata-action` 写入的构建时 commit），
再用 `git rev-list --count` 数本分支自那以后的新提交数。任何一步读不到都按「没发过」处理——
宁可多跑一次也不漏发。

拉取：

```bash
docker pull ghcr.io/jsntit/mihomo:v1.19.34-ts        # 多架构 index，按本机平台自动挑
```

除了多架构 tag，每个平台还有自己的独立 tag，在 GHCR 版本列表里一眼能看出是哪个平台，
也方便指定架构：

```bash
docker pull ghcr.io/jsntit/mihomo:v1.19.34-ts-amd64
docker pull ghcr.io/jsntit/mihomo:v1.19.34-ts-arm64
docker pull ghcr.io/jsntit/mihomo:v1.19.34-ts-armv7
docker pull ghcr.io/jsntit/mihomo:v1.19.34-ts-386
```

这些 tag 指向的就是 index 里那四个子 manifest，digest 与 index 内的完全一致（不是复制出来的新镜像）。

跑起来（配置目录挂进去，注意会盖掉镜像自带的 geo 文件，见下面注意事项）：

```bash
docker run -d --name mihomo --network host --cap-add NET_ADMIN \
  -v /opt/mihomo:/root/.config/mihomo \
  ghcr.io/jsntit/mihomo:v1.19.34-ts -d /root/.config/mihomo
```

两套镜像建议的 tag 方案：

| 来源 | version | latest |
|---|---|---|
| `Alpha-ts`（增强） | `v1.19.34-ts` | ✅ |
| `Alpha`（主线） | `v1.19.34` | ❌（或反过来，按你需要） |

## 流程说明

1. `binaries` job 按平台交叉编译，产出 `bin/mihomo-linux-<arch>-<版本>.gz` —— 命名严格对齐 `docker/file-name.sh`
   的约定（`amd64-v1` / `arm64` / `armv7` / `386` / `riscv64`），因为这些文件是 `Dockerfile` 用 `egrep` 挑出来的。
2. `image` job 用 `docker/file-name.sh` 反推每个平台期望的文件名做**自检**，缺一个平台就直接失败，
   避免推出一层缺架构的镜像。
3. `docker/build-push-action` 打多架构 index 推 GHCR，标签用 `GITHUB_TOKEN`（`packages: write`），
   不需要任何额外 secret；镜像 label 由 `docker/metadata-action` 自动指向本仓库。

## 注意事项

- **别手动跑上游那个 `Build` 工作流**：它末尾有个 `Docker` job 往 Docker Hub（`docker.io/jsntit/mihomo`）
  推镜像，依赖 `DOCKER_HUB_USER` / `DOCKER_HUB_TOKEN` 两个 secret，本 fork 没配，跑到那步会 login 失败。
  要出二进制用 **`Build (ts)`**，要出镜像用 **`Docker (GHCR)`**。
- **包可见性**：本仓库这套流程首次推送后，实测包是 `public`；若显示私有，去
  GitHub → 头像 → Your packages → `mihomo` → Package settings → Change visibility 改成 Public。
  私有包拉取需要 `docker login ghcr.io -u <user> -p <PAT with read:packages>`。
- **provenance / sbom 已关闭**：buildx 默认会给每个平台生成 attestation，在 index 里表现为
  `platform=unknown/unknown` 的额外 4 个条目，且都没有 tag，GHCR 版本列表里又是一串看不懂的
  `sha256:`。工作流里已设 `provenance: false` + `sbom: false`，要恢复构建溯源把这两行删掉即可。
- **`latest` 只有一个**：谁最后跑谁占。同时维护 ts 与主线两套镜像时，最好只给一套开 `tag_latest`。
  现在 tag 触发与手动都默认抢 `latest`，**定时巡检不抢**（分支自动构建给 `latest` 没意义）。
- **别用 `git describe --tags --always` 当版本号**：tag 之后分支再有提交，它返回
  `v1.19.34-ts-1-g5ed6d33` 这种「tag+N-ghash」形式 —— 这个字符串是**中间产物**，不是版本号。
  现在的取值顺序是：手动输入 → tag 触发时直接用 `github.ref_name` → 分支上取最近的 tag
  再拼 `-<ref>-<短sha>`（形如 `v1.19.34-ts-Alpha-ts-b7aff549`），并校验只含 `[A-Za-z0-9._-]`。
- **分支构建的版本号带 ref 与短 sha**：不同分支 / 不同提交各自出各自的镜像 tag，
  不会互相覆盖；代价是 tag 多，想清理就按「只保留带 `-ts` 的正式版」删。
- 镜像内含 `geoip.metadb` / `geosite.dat` / `geoip.dat`，是**构建时**从
  [meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 拉的最新版，配置目录是 `/root/.config/mihomo/`。
  用 `-v` 挂自己的目录会盖掉这三个文件，需要自己放一份。
- 镜像只提供 `linux/amd64`、`linux/arm64`、`linux/arm/v7`、`linux/386`、`linux/riscv64`；
  其它平台（armv5/armv6/mips/loong64 等）请用 Releases 里的静态二进制。
- Docker 构建需要 QEMU 模拟跑 `apk add`，4 平台大约 1~2 分钟；编译阶段每个平台约 1~2 分钟。

## 已发布记录

| 日期 | 版本 tag | 源 | 平台 | 镜像内自报版本 | 运行 |
|---|---|---|---|---|---|
| 2026-11-12 | `v1.19.34-ts` + `latest`（+ `-amd64` / `-arm64` / `-armv7` / `-386`） | `Alpha-ts` | amd64 / arm64 / arm/v7 / 386 | `Mihomo Meta v1.19.34-ts linux amd64 with go1.26.10` | [43144930113](https://github.com/jsntit/mihomo/actions/runs/43144930113) |
| 2026-11-12 | `v1.19.34`（+ `-amd64` / `-arm64` / `-armv7` / `-386`） | `Alpha` | amd64 / arm64 / arm/v7 / 386 | `Mihomo Meta v1.19.34 linux amd64 with go1.26.10`（`strings` 里无 `advertise-routes`，确认是纯净主线） | [43145585635](https://github.com/jsntit/mihomo/actions/runs/43145585635) |

平台 tag 指向的就是 index 里的子 manifest，digest 完全一致；GHCR 版本列表现在只剩 10 条，
全部带 tag，没有悬空的 `sha256:` 条目。

镜像层内的二进制逐个核过架构，不是把同一份 amd64 复制了四遍：

| 平台 | `file` 结果 |
|---|---|
| `linux/amd64` | ELF 64-bit LSB executable, x86-64, statically linked, stripped |
| `linux/arm64` | ELF 64-bit LSB executable, ARM aarch64, statically linked, stripped |
| `linux/arm/v7` | ELF 32-bit LSB executable, ARM, EABI5, statically linked, stripped |

## 运行验证

不只是 `-v` 打印版本，两个镜像里的二进制都实际启动跑过（从镜像层里解出 `/mihomo`，
用 `mixed-port: 27890` / `socks-port: 27891` / `external-controller: 127.0.0.1:29090` 启动）：

| 检查项 | ts 镜像（`v1.19.34-ts`） | 主线镜像（`v1.19.34`） |
|---|---|---|
| `GET /version` | `{"meta":true,"version":"v1.19.34-ts"}` | `{"meta":true,"version":"v1.19.34"}` |
| HTTP 代理出网 | `204`（gstatic generate_204，81ms） | — |
| SOCKS5 出网 | `200`（api.github.com，231ms） | — |
| 端口监听 | 27890 / 27891 / 29090 全部 LISTEN | — |

### ts 与主线是两套内核的差分证据

拿同一份带 ts 专属字段的配置分别喂给两个内核 `-t`（只校验配置、不联网）：

| 配置 | ts 内核 | 主线内核 |
|---|---|---|
| 合法：`advertise-routes: [192.168.99.0/24]` + `host-forward` | ✅ `test is successful` | ✅ `test is successful` |
| 非法 CIDR：`192.168.99.0/33` | ❌ `invalid tailscale advertise-routes prefix` | ✅ **仍然 successful** |
| 非 loopback 转发目标：`target: 192.168.1.5` | ❌ `host-forward.target must be a loopback IP` | ✅ **仍然 successful** |

结论：ts 内核真的在解析并校验这两个字段，主线内核**静默忽略**它们。
所以把 ts 的配置原样丢给主线镜像不会报任何错，只是那几行不生效 —— 这一点比报错更容易踩坑。

`strings` 计数同样对得上：`advertise-routes` ts 5 处 / 主线 0 处，`host-forward` ts 2 处 / 主线 0 处。

> `host-forward` 是 **map** 不是 list（`enabled` / `target` / `tcp` / `udp`），写成 list 会报
> `'host-forward' expected a map, got 'slice'`。

### riscv64

`linux/riscv64` 用 `push=false` 单独跑过一次（[43146967993](https://github.com/jsntit/mihomo/actions/runs/43146967993)）：
`编译 linux/riscv64` 与 `自检产物命名` 都通过（说明 `mihomo-linux-riscv64-<ver>.gz` 与
`docker/file-name.sh` 的期望一致），`登录 GHCR` 与 `给每个平台打独立 tag` 两步被 skip，
确认只构建、没往 registry 写任何东西。没有真正 push 是因为 riscv64 会单开一个 index，
混进现有 tag 会覆盖掉四平台的 index；要用的话单独换个版本号发。


