# rusty-v8-mobile：build workflow 操作手册

> 这份文档给将来接手维护这个 workflow 的 Claude Code session（或人）看。
> 目的：把我们 build 全部 rusty_v8 版本、排故、缓存、推进的方式固化下来，避免重走老路。

---

## 1. 背景

- 本仓库：公共 repo，基于 [denoland/rusty_v8](https://github.com/denoland/rusty_v8) 源码交叉编译 `librusty_v8.a` 并发布到 GitHub Releases。
- 目标平台：Android (aarch64 + x86_64) 和 iOS (device + simulator)。官方 rusty_v8 releases 不提供这些 target。
- 下游消费者通过 `RUSTY_V8_MIRROR=https://github.com/satouriko/rusty-v8-mobile/releases/download` 跳过本地 V8 编译。
- 目标：build 全部 rusty_v8 版本（130.0.7 → 147.1.0 及后续上游新版本）。

核心文件：
- `.github/workflows/build.yml` — 主 workflow，`workflow_dispatch` 触发
- `.github/workflows/sync.yml` — 自动跟踪上游（每天 08:00 UTC，按 `CUTOFF` 时间戳过滤后触发 build）

下游的链接问题（不在这里处理）：libc++ 符号冲突、`__clear_cache` 静态链接等由消费侧 cdylib 自己处理（`--allow-multiple-definition` + version script 隐藏 libc++ 符号、build.rs 静态链接 compiler-rt builtins）。

---

## 2. 工作方法

### 2.1 改 workflow 之前先**推演**

不是"改完跑一跑再说"。修改前先列出：

- 这次改动会影响哪些 target × 哪些 V8 版本？
- 当前在跑的 run 是不是和**驱动这次改动的根因**撞同一个问题？如果是，它们必定失败，取消省时间；如果不是（或还没跑到那一步），让它跑完。"因为我改了所以它会失败"是错的——我改东西只影响**之后**触发的 run。
- 下次 cold start 会走哪条 case？warm cache 会走哪条？
- 会不会污染其他 target 的 cache？

**踩过的坑**：未经推演就取消还在 Build V8 的 run、擅自在 job 没全绿时 release、同一个错误用同样的改动 retry 多次。

### 2.2 改动影响旧版本时要回测

workflow 的逻辑是全版本共享的——一次改动可能破坏老版本的 build。之前踩过：

- 从 crates.io 依赖改成 git clone 源码编译后，老版本（v130-v139）曾经能过 crates.io 但 git clone 后某些 transitive 依赖（`temporal_rs` 等）被 cargo 重 resolve 到不兼容版本
- src-cache key 的变更会让所有版本第一次 checkout 变冷起
- NDK / Clang / sysroot 之类的工具链版本升级可能兼容新 V8 但破坏老 V8

规则：

1. **改动前先想清楚影响面**：只影响新版本？影响所有版本？只影响某个 target？
2. **如果理论上可能破坏旧版本，主动挑一两个老版本回测**。挑策略：最早支持的版本 + 一个中间版本。不用全部测，但"理论可能受影响"时不能跳过
3. **回测不需要走完整流水线**——用现有的 cache，retry 几分钟就能看出是不是引入了新 error

不做回测而"先推新版本再说"会让老版本悄悄失修，以后要再 build 时发现断了已经说不清是什么改动破坏的。

### 2.3 巡检时审视三类信号，不只看红绿

1. **成败**：conclusion + 失败 step
2. **缓存状态**：哪些 cache hit / miss / 多大、**跨 retry 大小是否在增长**
3. **各 step 耗时**：和过去成功 run 比，某段莫名其妙变长说明有问题

**例子**：v8-deps cache 连续几次 retry 都停在 115MB 不增长，但正常应该是 300-400MB——说明 save 根本没跑（我们的 bug：`if: cache-hit != 'true'` 在首次 save 之后就永远满足 `cache-hit == true` 跳过 save）。partial cache 本身不是坏事，不涨才是坏事。

### 2.4 监控：gh watch + ScheduleWakeup 双保险

```bash
gh run watch <run_id> --interval 60 --exit-status   # 60s poll，完成时通知
```

再加一个 timer（15-30 min）做备用，防止 watch 静默失败。两种触发任一到来都继续工作，避免空等。

**重要：`--interval 60`**。默认 3s 太贪——一个 matrix run 每次 poll 取 run + jobs 约 2 个 API call，3s 间隔 = ~2400/h，两个 watch 并存 + 手动查询就能打到 5000/h 的 REST 上限。60s 间隔 = 120/h，两个 watch 也才 240/h，离上限远。V8 build 单步动辄几十分钟，60s 粒度完全够用。

60s 间隔下 API 用量稳低，无需刻意限制同时 watch 的数量（和并行 build 上限 2.6 节是不同的事）。但仍有几条纪律：

- **重挂 watch 前确认旧的 background task 已退出**，避免变僵尸叠加
- 主动查询批量取：`--json jobs` 一次拿全信息而不是分步
- 极端情况（误用默认 3s 或超多并发查询）打到 rate limit 要等 ~1h，期间所有 watch 会 403 退出，手动查询也废了

### 2.5 自主推进，别等提示

每次巡检完检查任务清单是否全部完成，没完成就**设置下次巡检**并**直接做下一步**。

### 2.6 并行上限 2 个 build

GitHub Actions macOS runner 限制 ~3-5 个，两个 version 并跑 = 4 个 macOS 作业，安全。超过会排队，反而变慢。

### 2.7 Release 纪律

`build` job 全绿才进 `release` job（workflow 已经 `needs: build` 保证）。
如果发现 release 被跳过却有 artifacts，不要手动上传——要么改 workflow 要么重跑。

---

## 3. 缓存策略

### 3.1 actions/cache 的坑（v4/v5 通用）

- `!path` exclusion pattern **对目录不可靠**。测试过 `!rusty_v8-src/third_party/rust-toolchain` 根本没排除掉，macOS 保存的 Darwin bindgen 照样进 cache，到 Linux 用就 Exec format error。正确做法：restore 后 `rm -rf`，save 前再 `rm -rf`。

- `actions/cache/save@v4` **同 key 静默跳过**（no-op，不报错）。要强制覆盖：先 `gh api -X DELETE "repos/.../actions/caches?key=X"`，再 save。需要 `permissions: actions: write`。

### 3.2 当前 key 约定

```
rusty-v8-src-v<version>                              # 源码（按版本，跨 target 共享）
android-ndk-r26c                                     # NDK（跨版本共享）
v8-deps-<target>-v<version>-api<api>                 # Android gn_out + clang
v8-deps-<target>-v<version>-ios<deployment_target>   # iOS gn_out + clang
```

固定 key，不带 run_id 后缀。save 前 API delete 强制刷新。per-(target, version) 隔离避免 matrix job 之间的竞态。

### 3.3 fail-fast 触发时 cache 完整度会分层

**状态：符合预期，不需要解决。** 4 个 matrix target，一个失败会 cancel 其他。每个 target 的 cache 完整度取决于当时它进行到哪：

| 结局 | gn_out cache |
|------|-------------|
| success | 完整 |
| failure after `[N/N] AR librusty_v8.a` | 完整（build 已完成，只是顶层 bindgen 炸）|
| failure mid-build | 部分 |
| cancelled mid-build | 部分（save 仍跑，但状态不完整）|

下次 retry 时完整 cache 的 target 秒过（~2min），部分 cache 的 ninja 重建几 min 到十几 min，仍远快于冷起的 60-110min。这是**期望行为**——失败 run 的残局也值得保存，迭代修到全绿。

**概率性遗留案例**：warm retry 下某一个 iOS target 偶尔完全无法利用 cache，耗时接近 cold build。已观察 2 次（2026-04-13 sim 43min、2026-04-14 device 70min），受影响 target 不固定。触发点已定位到 `rusty_v8-src/third_party/rust-toolchain/VERSION` 被 ninja 判 dirty（我们 `Purge OS-specific rust-toolchain` step 的必然副作用）；为何偶尔扩散到全图 rebuild 还未定位——需下次复现时靠 DIAG step 对比 `.ninja_log`。详见 [`docs/ios-warm-cache-rebuild-investigation.md`](./ios-warm-cache-rebuild-investigation.md)。

### 3.4 缓存信号诊断

partial cache 本身是功能（下次接着编），不是 bug。真正要警惕的是 **cache 内容该变不变** 或 **OS 错配** 这类具体问题。

- **cache 大小跨 retry 不增长**：save 逻辑有 bug（例如 `if: cache-hit != 'true'` 锁死、或 save 步骤被 cancel 前没跑到），不是 cache "残"的问题
- **cache 内容 OS 错配**：Darwin 二进制进了 Linux 的 cache → Exec format error。参见 4.1 rust-toolchain 的处理
- **ninja 重建步数 `[XX/YYYY]`**：YYYY 稳定（v140+ 大约 4239），XX 起点偏小说明 ninja 大部分认为 up-to-date，cache 复用到位
- **`already downloaded` vs `Downloading`**：判断 clang / rust-toolchain 是否复用

---

## 4. rusty_v8 交叉编译踩坑合集

### 4.1 rust-toolchain 是 OS-specific

`third_party/rust-toolchain/bin/` 里的 `bindgen`, `rustc` 等是 V8 自己下载的、和 OS 绑定的二进制。
**不能**跨 OS 共享。src cache save 前必须 `rm -rf`。

### 4.2 V8 internal bindgen vs rusty_v8 顶层 bindgen

**两个完全独立**的 bindgen 调用，配置互不相通：

| | 触发时机 | libclang 来源 | 参数来源 |
|--|---------|-------------|---------|
| V8 internal | ninja 调用，build V8 中途 | `third_party/rust-toolchain/bin/bindgen`（V8 自带）| CLI 参数 `--libclang-path <V8 自带的 clang>` |
| rusty_v8 顶层 | `cargo build` 末尾，V8 build 完之后 | 从 env `LIBCLANG_PATH` 找 | bindgen crate 读 `BINDGEN_EXTRA_CLANG_ARGS[_<target>]` |

V8 internal 不读 env `LIBCLANG_PATH`（用 CLI），但 **读** `BINDGEN_EXTRA_CLANG_ARGS_<target>`（bindgen CLI 和 crate 都读）。
所以 env 里加 `--target=...` 会污染 V8 internal 对 host 工具（`clang_x64_for_rust_host_build_tools`）的调用——报 `-msse3 unsupported` 之类。

**解法**：v140+ Android 的 env 只塞 `--sysroot` + `-isystem ...`，**不塞 `--target`**。bindgen crate 会从 `TARGET` 自动推；V8 internal CLI 保留自己的 `--target=x86_64-linux-gnu`（host）或 `<target><api>`（target）不被覆盖。

### 4.3 V8 版本之间的破坏性变化

```
v130-v139:
  NDK libc++ 和 Chromium libc++ 还兼容 → -isystem NDK c++/v1 能用
  env BINDGEN_EXTRA_CLANG_ARGS 可以带 --target（没有 V8 internal bindgen 会被污染）
  LIBCLANG_PATH=/usr/lib/x86_64-linux-gnu

v140+:
  V8 引入 internal ninja bindgen → env 不能带 --target
  Chromium 自带 libc++ 加入构建，要求 Clang 19+
  LIBCLANG_PATH 要 Clang 19+ 的（apt.llvm.org 装 llvm-19）
  -isystem 改用 V8 自带的 Chromium libc++：$V8_SRC/third_party/libc++/src/include

v142+:
  Chromium libc++ bump，代码里用 __builtin_clzg / __builtin_ctzg（Clang 19 才加）
  → NDK r26c 的 Clang 17 libclang 完全不认
  → 必须用 llvm-19（v140+ 的方案正好覆盖）

v145+:
  host rust toolchain 依赖 Debian bullseye sysroot 才能过 gn gen 的 assert
  → 在 Prepare Android NDK 里跑 build/linux/sysroot_scripts/install-sysroot.py --arch=amd64
```

**iOS 不受以上影响**：v140+ 只靠 Xcode clang，LIBCLANG_PATH 不设 / 用默认就能过；v130-v139 需要塞 `-isysroot $SDK --target=$CLANG_TARGET`（`xcrun --sdk iphoneos/iphonesimulator --show-sdk-path`）。bindgen 0.70.1 对 iOS sim 的 target triple 有 bug（`arm64-apple-ios-sim` 而不是正确的 `arm64-apple-ios-simulator`），需要在 env 里显式 override——这是 v130 iOS sim 的专属问题。

### 4.4 其他 V8 build.rs 需要补的文件

rusty_v8 checkout 时精简掉了 V8 期望存在的一些文件，workflow 手动造：

```
build/android/*.pydeps                           # pydeps stubs
build/android/pylib/results/presentation/...pydeps
build/android/test_wrapper/...pydeps
build/rust/known-target-triples.txt              # v140+
build/linux/debian_bullseye_amd64-sysroot/       # v145+，用 install-sysroot.py 下
```

以及 `.cargo/config.toml` 里的 linker（NDK 的 clang wrapper，按 target 选 aarch64- 或 x86_64- 前缀）。

---

## 5. 排查错误的套路

### 5.1 看 panic 位置，别猜

```
thread 'main' panicked at rusty_v8-src/build.rs:233:6
```

`build.rs:233` 是 **顶层** bindgen 生成位置。log 里如果前面有 `[4239/4239] AR obj/librusty_v8.a`，说明 V8 编译已经成功，只是顶层 bindgen 炸。

vs

```
ERROR at //build/config/sysroot.gni:60:7
```

这是 gn gen 阶段失败，还没进 ninja——V8 internal 就起不来。

### 5.2 看 `-isystem` 路径里出错的文件

```
third_party/libc++/src/include/...              → Chromium 新 libc++（V8 自带）
android_ndk/.../sysroot/usr/include/c++/v1/...  → NDK 的旧 libc++
/usr/include/c++/11/...                         → Linux host libstdc++
```

同时包含两个 libc++ → 打架。想清楚要哪一份、把另一份从 include path 拿掉。

### 5.3 对比成功 vs 失败 run

相同 V8 版本、相同 workflow 有时一次成一次败——看 log 对比 cache hit 状态、step 耗时、env 变量。常见原因：cache 内容 OS 错配、runner 调度噪声（macOS 特别明显）、网络抖动。

---

## 6. 当前状态（快照）

**本节是时间点快照，不保证和仓库一致。**
权威状态：`gh release list` + `gh run list`。

至 2026-04-13：
- ✅ 130.0.7 / 134.5.0 / 135.1.1 / 136.0.0 / 137.3.0 / 139.0.0 / 140.2.0 / 142.2.0 / 145.0.0 / 146.9.0 / 147.1.0 全部 released
- ✅ `sync.yml` 已启用（daily cron 08:00 UTC 检查上游新版本并触发 build.yml）
- 遗留调查项：iOS-sim 在 warm-cache retry 下偶尔异常慢（见 3.3 节）

---

## 7. 失败经验（做什么 → 得到什么教训）

| 做过的蠢事 | 教训 |
|-----------|-----|
| 没推演就 cancel 还在 Build V8 的 run | 猜失败不算数，除非推演证明必定失败 |
| `|| true` 吞掉 Build V8 的错误码 | CI 失败必须可见，不加 fallback |
| src-cache key 带 runner.os | 源码 OS 无关，加上这一维没意义，反而分裂 cache |
| `!path` exclusion 信了没验证 | 先验证，别信广告 |
| run_id 后缀 key 强行覆盖 | 造成 cache 积累 LRU 挤掉其他 cache，API DELETE 就够 |
| 一次同时推 4 个版本 | macOS runner 排队，总耗时反而更长。控制在 2 |
| 半成品 cache 被 cache-hit 逻辑锁死 | 不要 `if: cache-hit != 'true'` 守 save，要 force refresh |
| 同一个 bindgen 错误改 env var 配置 | 先搞清是哪个 bindgen（V8 internal / 顶层）失败，对症下药 |
| 用 runner 架构理论猜 iOS-sim 比 iOS device 慢 | 历史数据显示冷起 iOS-sim 通常最快，warm cache 才偶尔反转。用数据不用直觉 |
