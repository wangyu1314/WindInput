# 从 Linux 发版：流程与检查点

> **状态（2026-09-13）**：签名段已端到端实测通过；**tag/push 段与 sign-draft 段尚未在
> Linux 上完整跑过**，本文的命令由 `scripts/release.ps1` 逐段翻译而来。第一次照此发版
> 请逐步确认，不要挂上 `--yes` 一把梭。哪些验过、哪些没验，每节都标了。

开发主力迁到 Linux 后，`scripts/release.ps1` 跑不了了（它是 PowerShell，且 `sign-draft`
段要调 `dev.ps1`）。本文记录在 Linux 上手工完成一次发版的完整序列，供日后固化成
`scripts/release.sh`。

---

## 0. 本文的占位变量

下面所有命令里的 `$BUILD_HOST` / `$BUILD_ROOT` 都来自 **`scripts/build.local`**
（gitignore，模板见 `build.local.example`）—— 机器地址属于各人的环境，不入库。

```bash
cd ~/develop/windinput/WindInput
source scripts/build.local                 # 提供 WIND_BUILD_REMOTE / WIND_BUILD_ROOT
BUILD_HOST="$WIND_BUILD_REMOTE"            # 形如 user@host
BUILD_ROOT="$WIND_BUILD_ROOT"              # 编译机上的仓根，形如 X:/path/WindInput
```

---

## 1. 三机分工与不可替代性

| 机器 | 角色 | 不可替代的原因 |
|---|---|---|
| Linux（本机） | 代码、git、触发 CI、上传 Release | 主力开发机 |
| 编译机（Windows） | **签名** | 云签名客户端与 `signtool` 只有 Windows 有 |
| GitHub CI | Windows + **macOS** 产物 | **macOS `.pkg` 只能在 `macos-14` runner 上产**（通用二进制 arm64+x86_64） |

⚠️ **CI 不能跳过**。VM 能编译能签名，看起来可以绕过 CI 直接出包 —— 但那样**拿不到 macOS
包**。发版是 Windows + macOS 两套产物，缺一不可 —— 「发布包静默少组件」是踩过的坑
（macOS pkg 曾整版缺设置 app，全程无报错）。

签名只覆盖 **Windows 的 4 个资产**：

```
WindInput-Setup-<版本>.exe          + .sha256
WindInput-Portable-<版本>.zip       + .sha256
```

macOS 的 `.pkg` 由 CI 直接传进 Release，**不经过本流程**。

---

## 2. 五仓的推送顺序（顺序错了会静默发出错误的包）

| 仓 | 打 tag | 说明 |
|---|---|---|
| `wind-ui-rust` | ✗ 只推送 | 是 `wind-setting` 的 path 依赖（`windui = { path = "../wind-ui-rust" }`），必须先到位。`wind-installer` 走 crates.io 的 `windui = "0.8"`，不受影响 |
| `wind-setting` | ✓ | |
| `wind-portable` | ✓ | |
| `wind-installer` | ✓ | |
| **`WindInput`** | ✓ | **永远最后推** |

★ **为什么主仓最后**：推 `WindInput` 的 `v*` tag 会**立刻**触发 `.github/workflows/release.yml`，
而该 workflow 用 `actions/checkout` 拉附属仓时**没有指定 ref**，取的是它们默认分支的最新
提交。附属仓没先到位，CI 就会拿旧代码构建出错误的发布包 —— 且全程不报错。

---

## 3. 版本号：tag 是唯一真源

`release.yml` 在 tag 触发时执行 `V="${GITHUB_REF_NAME#v}"` 并写入 `docs/VERSION`，**CI 完全
以 tag 名为准**。`docs/VERSION` 只用于两处：本地 `dev.ps1`/`dev.sh` 构建，以及
`workflow_dispatch` 手动触发时的开发占位版本。

⇒ **发版不需要为改版本号单独提交 commit**，bump 只是选一个新 tag 名。

⚠️ 但 **`unstage` 有版本硬校验**（`stage.json` 的版本 vs 本机 `docs/VERSION`），所以签名段
之前必须把执行签名那台机器的 `docs/VERSION` 对齐到 tag 版本。这是那步唯一会静默出坏包的
地方，被刻意拦死了 —— 详见 [code-signing.md](code-signing.md) 4.1。

---

## 4. 发版前置检查

**✅ 以下各项均可在 Linux 上执行。**

```bash
cd ~/develop/windinput

# 1. 五仓都在目标分支、工作区干净、无未推送提交
for r in wind-ui-rust wind-setting wind-portable wind-installer WindInput; do
    printf '%-16s %s\n' "$r" "$(git -C $r status --short --branch | head -1)"
    git -C $r status --short | grep -q . && echo "    ⚠️ 工作区有改动"
done

# 2. gh 已登录
gh auth status

# 3. VM 可达
ssh "$BUILD_HOST" 'pwsh -NoProfile -Command "whoami"'
```

**检查点**：

- 每个仓都应是 `## main...origin/main`，**没有 `[ahead N]`**。有 `ahead` 说明有未推送提交 ——
  ⚠️ **先确认那些提交是不是你的**。本工作区常有并发会话，把别人未完成的工作 tag 进发版是
  不可逆的。
- ⛔ `repo sync` 之后会停在**游离 HEAD**，此时绝不能发版。先 `git status -sb` 确认在分支上。

### ★ 时序：先开签名会话，再打 tag

签名会话有 **2 小时**时限，CI 约 **20 分钟**。正确顺序是**先在 VM 桌面登录会话**，再触发
发版 —— 等 CI 跑完时会话仍在有效期内。

```bash
# 人工：在 VM 桌面打开签名客户端，手机二次验证登录
# 然后从 Linux 确认会话真的可用：
ssh "$BUILD_HOST" "pwsh -NoProfile -Command \"Set-Location '\$BUILD_ROOT'; .\\scripts\\sign.ps1 -Status\""
```

**检查点**：输出应有 `会话     : 可用`。

⚠️ **`HasPrivateKey=True` 不是判据** —— 云签名私钥在服务商 HSM 里，会话没建立时这个属性
照样是 True。只有 `-Status` 报「可用」或真签出一个文件才算数。

---

## 5. 打 tag 与推送

**⚠️ 未在 Linux 实测**（逐段译自 `release.ps1` 的 `Invoke-Release`）。

```bash
cd ~/develop/windinput
V=0.121.3                      # 新版本号，不带 v
BR=main

# 1. 附属仓：先推 wind-ui-rust（不打 tag）
git -C wind-ui-rust push origin $BR

# 2. 三个附属仓：打 tag 并推
for r in wind-setting wind-portable wind-installer; do
    git -C $r tag -a "v$V" -m "v$V" || { echo "$r tag 失败"; break; }
    git -C $r push origin $BR
    git -C $r push origin "v$V"
done

# 3. 主仓最后（推 tag 会立刻触发 CI）
git -C WindInput push origin $BR
git -C WindInput tag -a "v$V" -m "v$V"
git -C WindInput push origin "v$V"
```

**检查点**：

- 附属仓的 tag **必须在主仓 tag 之前**到达远端（见第 2 节）。
- 推完主仓 tag 后 CI 应在 1 分钟内起来：
  `gh run list --workflow release.yml --limit 3 -R huanfeng/WindInput`

⚠️ 仓库可能装有 **pre-push hook**（`wind-ui-rust` 会跑 clippy + 全量测试），推送会变慢。
`release.ps1` 对此有专门处理，手工推时耐心等即可，别用 `--no-verify` 绕过。

---

## 6. 等 CI

**⚠️ 未在 Linux 实测。**

```bash
cd ~/develop/windinput/WindInput
gh run list --workflow release.yml --branch "v$V" --limit 5
gh run watch <runId>           # 或轮询 gh run view <runId>
```

**检查点**：`build-windows` 与 `build-macos` **两个 job 都要成功**。CI 约 20 分钟。

产出的 artifact：

| artifact | 内容 | 用途 |
|---|---|---|
| `stage-windows` | `WindInput-Stage-<版本>.zip`（`build/` 散件 + 安装器三件套） | **签名段要用这个** |
| `dist-windows` | CI 自己打的未签名 Setup/Portable | 不用，会被签名版覆盖 |
| `dist-macos` | `.pkg` + `.sha256` | CI 直传 Release，不经本流程 |

⚠️ **artifact 保留期 14 天**，过期只能重跑 CI。

---

## 7. 签名段：委托 VM

**✅ 本节的签名部分已实测通过（2026-09-13，v0.121.2）**，只是当时的产物来自本地编译而非
CI artifact；下载与回传两步未实测。

```bash
cd ~/develop/windinput/WindInput
TMP=$(mktemp -d)

# 1. 找到本次 tag 的 run，拉中转产物
RUN=$(gh run list --workflow release.yml --branch "v$V" --limit 10 \
        --json databaseId,conclusion -q '[.[]|select(.conclusion=="success")][0].databaseId')
gh run download "$RUN" --name stage-windows --dir "$TMP"
ls -la "$TMP"/WindInput-Stage-*.zip

# 2. 传到 VM
scp "$TMP"/WindInput-Stage-*.zip "$BUILD_HOST":"$BUILD_ROOT/dist/"

# 3. 对齐 VM 的 docs/VERSION（unstage 的硬校验要它等于 tag 版本）
printf '%s\n' "$V" > /tmp/VERSION
scp /tmp/VERSION "$BUILD_HOST":"$BUILD_ROOT/docs/VERSION"
```

⚠️ **同步 VERSION 必须用 `scp`，别拼 PowerShell 写文件** —— bash 单引号里的 `\"` 和 `''`
会原样传过去，实测连踩两次转义坑。

★ **一切要在 VM 上跑的 PowerShell，都写成本地脚本再 `scp` 过去执行**，不要往
`ssh '...'` 里塞多层引号 —— bash 的单/双引号与 PowerShell 的转义叠在一起极易出错，
且错法是「悄悄传了个字面量过去」，不报错。

```bash
# 4. VM 上还原 + 签名 + 验签
cat > /tmp/do-sign.ps1 <<PS1
\$ErrorActionPreference = "Stop"
Set-Location "$BUILD_ROOT"
.\scripts\dev.ps1 unstage ".\dist\WindInput-Stage-$V.zip"; if (\$LASTEXITCODE) { exit 1 }
.\scripts\dev.ps1 sign 8s;                                   if (\$LASTEXITCODE) { exit 1 }
.\scripts\dev.ps1 sign 9s;                                   if (\$LASTEXITCODE) { exit 1 }
.\scripts\dev.ps1 verify-sign;                               if (\$LASTEXITCODE) { exit 1 }
PS1
scp /tmp/do-sign.ps1 "$BUILD_HOST":"$BUILD_ROOT/../do-sign.ps1"
ssh "$BUILD_HOST" "pwsh -NoProfile -File '$BUILD_ROOT/../do-sign.ps1'"

# 5. 回传 4 个签名资产
scp "$BUILD_HOST":"$BUILD_ROOT/dist/WindInput-Setup-$V.exe*" dist/
scp "$BUILD_HOST":"$BUILD_ROOT/dist/WindInput-Portable-$V.zip*" dist/
```

**检查点（逐条都要看，脚本报「完成」不算数）**：

1. `unstage` 打印 **两行** `→ ...`：`build\` 和 `target\release (安装器三件套)`。只有第一行
   说明三件套没还原，`pack.ps1` 会现编。
2. `sign 8s` 应出现 **三段签名**：`5 个文件` → `1 个文件`(uninstall.exe) → `1 个文件`(Setup.exe)。
   **共 7 次配额**，不是 6 次。
3. `sign 9s` 应报 `0 个已签, 5 个跳过` —— 按签名者指纹识别为已签，不重复扣配额。跳过数
   不是 5 就要查。
4. `verify-sign` exit 0。⚠️ 它**扫 `dist\` 顶层且不递归**，历史遗留的未签名旧包会把退出码
   拖成 1；把旧包移进 `dist\_archive\` 即可。
5. ⚠️ **`verify-sign` 验不到便携包内部和时间戳**。zip 签不了（Authenticode 只认 PE），
   它绿了只代表 `Setup.exe` 没问题。便携包要单独验：

同样写成脚本传过去：

```bash
cat > /tmp/verify-portable.ps1 <<PS1
\$ErrorActionPreference = "Stop"
\$st = (Get-ChildItem "C:\Program Files (x86)\Windows Kits\10\bin" -Recurse -Filter signtool.exe |
         Where-Object { \$_.FullName -match "\\x64\\" } | Sort-Object FullName -Descending)[0].FullName
\$tmp = Join-Path \$env:TEMP "wind-verify-portable"
if (Test-Path \$tmp) { Remove-Item \$tmp -Recurse -Force }
Expand-Archive "$BUILD_ROOT/dist/WindInput-Portable-$V.zip" \$tmp -Force
Get-ChildItem \$tmp -Recurse -File | Where-Object { \$_.Extension -in @(".exe",".dll") } | ForEach-Object {
    \$o  = & \$st verify /pa /v \$_.FullName 2>&1
    \$ts = (\$o | Where-Object { \$_ -match "timestamped" }) -replace ".*timestamped:\s*", ""
    "{0,-20} {1}" -f \$_.Name, \$(if (\$ts) { "时间戳 " + \$ts.Trim() } else { "★无时间戳★" })
}
Remove-Item \$tmp -Recurse -Force
PS1
scp /tmp/verify-portable.ps1 "$BUILD_HOST":"$BUILD_ROOT/../verify-portable.ps1"
ssh "$BUILD_HOST" "pwsh -NoProfile -File '$BUILD_ROOT/../verify-portable.ps1'"
```

**检查点**：5 个 PE 每个都要有 `The signature is timestamped: ...`。
★ **时间戳不可省略** —— 没有它，签名会在证书到期当天集体失效，连早已发出去、用户机器上
装着的包也会一起变成「未知发布者」。

---

## 8. 上传 Release 并去掉未签名横幅

**⚠️ 未在 Linux 实测。**

```bash
cd ~/develop/windinput/WindInput
gh release upload "v$V" \
    "dist/WindInput-Setup-$V.exe" \
    "dist/WindInput-Setup-$V.exe.sha256" \
    "dist/WindInput-Portable-$V.zip" \
    "dist/WindInput-Portable-$V.zip.sha256" \
    --clobber

# 正文里 CI 加的未签名横幅要摘掉
gh release view "v$V" --json body -q .body > /tmp/body.md
# 编辑 /tmp/body.md 删掉未签名横幅段落
gh release edit "v$V" --notes-file /tmp/body.md
```

⚠️ **只覆盖草稿 Release**。覆盖**已发布**的 Release 会让已下载用户的 sha256 对不上，且
`release-published.yml` 早已按旧文件同步到 R2、那边的 `latest.json` 也指向旧 hash。
`release.ps1` 对此默认拒绝，要 `-Force` 才继续 —— 手工操作时请自行守住这条。

**检查点**：`gh release view "v$V"` 确认 4 个 Windows 资产 + macOS `.pkg` 齐全。
⚠️ 少组件是静默的，数一遍。

---

## 9. 失败恢复

| 情况 | 处理 |
|---|---|
| 签名产物已出、上传失败 | **别重跑签名段**（会重新拉产物、重签一遍，白扣 7 次配额）。产物就在 VM 的 `dist\`，回传后直接 `gh release upload --clobber` 补传 |
| 会话在 CI 跑完前过期 | 重新登录会话，从第 7 节开始即可，前面的 tag/CI 不用重来 |
| artifact 过期（>14 天） | 只能重跑 CI：`gh run rerun <runId>` |
| 附属仓 tag 顺序推错了 | 删远端 tag 重推代价高；更稳的是**换一个新版本号重发**，tag 是廉价的 |

---

## 10. 与 `release.ps1` 的对应关系

| `release.ps1` | 本文 | Linux 可否原生 |
|---|---|---|
| `status` / `check` | 第 4 节 | ✅ 纯 git |
| `patch` / `minor` | 第 5 节 | ✅ 纯 git |
| （等 CI） | 第 6 节 | ✅ `gh` |
| `sign-draft` | 第 7–8 节 | ⚠️ 签名段必须委托 VM，其余 `gh` 可做 |
| `auto-sign` / `-AutoSign` | — | 尚未有对应物 |

固化成 `scripts/release.sh` 时，人工不可消除的只有一步：**在 VM 桌面登录签名会话**
（手机二次验证 + GUI 客户端，2 小时时限）。其余全部可自动化。

相关：[code-signing.md](code-signing.md)（签名原理、五个接线点、4.3 签名机迁到编译机）
