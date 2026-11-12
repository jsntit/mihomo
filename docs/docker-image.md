# Docker 镜像（GHCR）

仓库自带上游的 `Dockerfile` + `docker/file-name.sh`，原本只给上游发布流水线用（推到 Docker Hub）。
`.github/workflows/docker.yml` 复用了这两个文件，把镜像发到你自己的 GitHub Container Registry。

## 用法

两种触发方式：

1. **手动**：Actions → **Docker (GHCR)** → Run workflow，按下表填参数。
2. **自动**：push 一个 `v*-ts` 的 tag，就会跟着 `Build (ts)` 一起出镜像 ——
   此时 `ref` 取该 tag、`version` 取 tag 名（`git describe` 的结果正好是 tag）、`tag_latest=true`。

| 参数 | 默认 | 说明 |
|---|---|---|
| `ref` | `Alpha-ts` | 打包哪个分支/tag。`Alpha-ts` = 带 Tailscale 增强；`Alpha` = 纯净主线 |
| `version` | 空 | 写进内核 `Version` 与镜像 tag；留空则用 `git describe` |
| `platforms` | `linux/amd64,linux/arm64,linux/arm/v7,linux/386` | 可加 `linux/riscv64`。只用 `docker/file-name.sh` 支持的这五个 |

| `tag_latest` | `true` | 额外打 `latest`。**两套镜像同时维护时，只给其中一套开** |
| `push` | `true` | 关掉则只构建不推送，用于验证 |

拉取：

```bash
docker pull ghcr.io/jsntit/mihomo:v1.19.34-ts
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

- **包可见性**：本仓库这套流程首次推送后，实测包是 `public`；若显示私有，去
  GitHub → 头像 → Your packages → `mihomo` → Package settings → Change visibility 改成 Public。
  私有包拉取需要 `docker login ghcr.io -u <user> -p <PAT with read:packages>`。
- **index 里会多出 4 个 `unknown/unknown` 条目**：那是 buildx 默认生成的 provenance 证明（每个平台一个），
  不是多出来的架构，`docker pull` 会按本机平台自己挑。要彻底干净可以在
  `docker/build-push-action` 里加 `provenance: false`（代价是失去构建溯源）。
- **`latest` 只有一个**：谁最后跑谁占。同时维护 ts 与主线两套镜像时，最好只给一套开 `tag_latest`。
- 镜像内含 `geoip.metadb` / `geosite.dat` / `geoip.dat`，是**构建时**从
  [meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 拉的最新版，配置目录是 `/root/.config/mihomo/`。
  用 `-v` 挂自己的目录会盖掉这三个文件，需要自己放一份。
- 镜像只提供 `linux/amd64`、`linux/arm64`、`linux/arm/v7`、`linux/386`、`linux/riscv64`；
  其它平台（armv5/armv6/mips/loong64 等）请用 Releases 里的静态二进制。
- Docker 构建需要 QEMU 模拟跑 `apk add`，4 平台大约 1~2 分钟；编译阶段每个平台约 1~2 分钟。

## 已发布记录

| 日期 | 版本 tag | 源 | 平台 | 镜像内自报版本 | 运行 |
|---|---|---|---|---|---|
| 2026-11-12 | `v1.19.34-ts` + `latest` | `Alpha-ts` | amd64 / arm64 / arm/v7 / 386 | `Mihomo Meta v1.19.34-ts linux amd64 with go1.26.10` | [43141749531](https://github.com/jsntit/mihomo/actions/runs/43141749531) |
| 2026-11-12 | `v1.19.34` | `Alpha` | amd64 / arm64 / arm/v7 / 386 | `Mihomo Meta v1.19.34 linux amd64 with go1.26.10`（`strings` 里无 `advertise-routes`，确认是纯净主线） | [43142886442](https://github.com/jsntit/mihomo/actions/runs/43142886442) |

