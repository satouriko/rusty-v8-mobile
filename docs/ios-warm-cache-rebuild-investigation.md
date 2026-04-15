# iOS warm-cache catastrophic rebuild 调查

> 活跃调查。目标：找到为何某次 warm retry 下某一个 iOS target 会
> 完全无法利用 cache，耗时等同 cold build。

## 1. 症状

- 概率性触发。4 次后续采样（见 §3）都没复现。
- warm retry 下**某一个** iOS target 的 `cargo build` 耗时接近 cold
  build（60–70min），其他 3 个 target 正常 warm 命中（~2min）。
- 受影响 target 不固定：观察到一次落在 `sim`，一次落在 `device`。
- cache restore 本身正常（大小、hit 状态都对），不是 restore 失败。
- `cargo build` 输出期间完全静默——`Compiling v8` 之后直到 build.rs
  末尾的 `warning: Not using sccache` 之间没有任何 ninja 进度输出
  （ninja 的 stdout 被 build.rs 截流了，不是真卡住）。

## 2. 已观察样本

| 日期 | V8 | run | 慢 target | 慢耗时 | 同 run 其他 iOS | 备注 |
|-----|-----|-----|---------|-------|----------------|-----|
| 2026-04-13 | 142.2.0 | `24320385806` | `aarch64-apple-ios-sim` | 43.5min | `ios` 1.7min | 父 run `24318352657`，两者都 success、cache 各 264MB |
| 2026-04-14 | 147.1.0 | `24409852097` | `aarch64-apple-ios` | 70min (4185s) | `ios-sim` 4min (255s) | 父 cold run `24323167710` 的 `ios` 耗时 4151s → warm retry **几乎完全没加速** |

2026-04-14 这次是关键证据：warm retry 4185s vs 同 target cold 4151s，
说明 ninja 确实把整个图都当成 dirty 重做了，不是部分加速后的噪声。

## 3. 对照快 case（有 DIAG 数据）

2026-04-15 起在 `build.yml` 加了 iOS 专用的诊断 step（`DIAG pre-build
snapshot` / `DIAG post-build snapshot`）。在 cache-hit 的 iOS job 上
dump `args.gn` / `toolchain.ninja` / Xcode & SDK 路径 / ninja dry-run
explain / post-build `.ninja_log`。

4 次 warm retry 都是快 case（2-3min）：

| run | v147.1.0 iOS device | iOS sim |
|-----|--------------------|--------|
| `24458738289` | 2min | 2min |
| `24459169428` | 2min | 2min |
| `24459694967` | 2min | 2min |
| `24459868634` | 2min | 2min |

快 case 的 DIAG 显示：

```text
ninja: Entering directory `v8-build/.../gn_out'
ninja explain: rusty_v8-src/third_party/rust-toolchain/VERSION is dirty
[0/1] Regenerating ninja files
```

`.ninja_log` 只有约 15 行新条目，全是 `obj/v8/v8_initializers/*.o` ——
说明 GN regen 后仅让 `v8_initializers` 这一组依赖 rust-toolchain 的
target 失效，其余 4200+ target 保持 cached。ninja 做了 ~45s 活，整个
job 2min 完。

### 快 case 的 args.gn（warm restore 版，仅 9 行）

```
is_debug = false
use_custom_libcxx = true
v8_enable_sandbox = false
v8_enable_pointer_compression = false
v8_enable_v8_checks = false
rusty_v8_enable_simdutf = false
host_cpu = "arm64"
clang_base_path = "/Users/runner/work/rusty-v8-mobile/rusty-v8-mobile/v8-build/target/<target>/release/clang"
target_cpu = "arm64"
```

post-build 的 args.gn **内容完全一致**——所以至少快 case 下 GN regen
没有修改 args。

## 4. 已定位的触发链

**触发点确定**：`rusty_v8-src/third_party/rust-toolchain/VERSION` 被
ninja 视为 dirty。

直接原因：workflow 里的 **`Purge OS-specific rust-toolchain`** step
（`build.yml:105-106`）在 src-cache restore 后无条件 `rm -rf
rusty_v8-src/third_party/rust-toolchain`，处理 §4.1 的 OS-绑定 bindgen
问题。V8 build.rs 后续重新下载这个目录，重下后 `VERSION` 文件 mtime
是新的（哪怕内容一样）。ninja 只看 mtime，判定 dirty → 让 GN 重跑
→ 重建 ninja 文件。

**快 case**：GN regen 产出的 `build.ninja` / `toolchain.ninja` 内容
和 restore 出来的版本**完全一致**（GN 是确定性的，输入没真变化），
mtime 更新但内容校验发现一致 → ninja 只 invalidate 直接依赖 VERSION
的 ~15 个 target，其余沿用缓存。

## 5. 未解开的部分（慢 case 才能回答）

快 case 路径已清楚。慢 case（未复现，DIAG 未抓到）的开放假设：

1. **GN regen 写出不同内容的 `toolchain.ninja`**
   - `toolchain.ninja` 的 rule 命令行里带 `TOOL_VERSION=<unix-ts>`，
     理论上每次 gn gen 刷新成当前时间戳
   - 快 case 观察到的 `TOOL_VERSION=1776048109`（= 2026-04-13T02:41:49Z）
     对应 v147.1.0 **原始 cold build** 时间——说明快 case 里 GN
     没真的重写 toolchain.ninja（否则 TOOL_VERSION 应变成 2026-04-15）
   - 慢 case 可能：某些情况下 GN gen 确实重写了 toolchain.ninja，
     TOOL_VERSION 刷新 → 所有 rule command 字面不同 → ninja 判所有
     target dirty → 全图重建
   - 触发条件未知，可能与并发 / 文件系统时序有关

2. **`.ninja_deps` 和 build.ninja 不一致**
   - ninja 的二进制 dep 数据库可能某些情况下被认为 stale
   - 一旦 ninja 放弃 dep 信息，会保守 rebuild 大量 target

3. **macOS 文件系统 mtime 精度 / APFS snapshot 干扰**
   - macOS 的 gtar 恢复 mtime 依赖文件系统粒度
   - 某些 runner 的 APFS 在高并发下可能让 mtime 比较偏差

## 6. 下次撞上慢 case 的排查 SOP

**判定慢 case**：`gh run view <id>` → 某一 iOS target 的 Build V8 step
耗时 > 10min 即触发排查。

job 跑完后（`if: always()` 保证 post-build DIAG 会跑，即使 build 阶
段超时 cancel，需要 DIAG 已执行）：

```bash
# 1. 抓慢 job 的完整日志
gh api "repos/satouriko/rusty-v8-mobile/actions/jobs/<jobid>/logs" > /tmp/slow.log

# 2. 定位关键 section
awk '/DIAG pre-build/,/DIAG post-build/' /tmp/slow.log > /tmp/slow-diag.log

# 3. 与任意一次快 case 的 DIAG 对比（可以拿 24458738289 的 ios-device
#    job 71466623323 作基线）
gh api "repos/satouriko/rusty-v8-mobile/actions/jobs/71466623323/logs" > /tmp/good.log
awk '/DIAG pre-build/,/DIAG post-build/' /tmp/good.log > /tmp/good-diag.log
diff /tmp/good-diag.log /tmp/slow-diag.log
```

**关键对照项**（按优先级）：

| 项 | 快 case | 慢 case 预期 | 含义 |
|----|---------|-------------|-----|
| `.ninja_log` 行数 | ~15 | 几千 | rebuild 范围 |
| `.ninja_log` 覆盖目录 | 仅 `v8_initializers/*` | 所有 `v8/*`、`v8_libbase/*`、`torque/*` 等 | invalidate 是否全图 |
| pre vs post `args.gn` | 字节一致 | 可能一致、也可能差空白/顺序 | 看 GN 是否"真的"重写 args |
| `toolchain.ninja` 里的 `TOOL_VERSION` | 1776048109（原始 cold 时间戳）| 可能刷新到 warm retry 时间戳 | GN 是否重写 toolchain.ninja |
| ninja dry-run explain 输出 | 只报 VERSION dirty | 可能报更多路径（哪个文件 + 为何 dirty）| 一手证据 |
| Xcode 活跃版本（`xcode-select -p`）| `Xcode_16.4.app` | 可能不同 | 若不同则 SDK 路径变动 |
| `/Applications/Xcode.app` symlink 目标 | `Xcode_16.4.app` | 可能不同 | 同上 |

**如果确认是 TOOL_VERSION 刷新导致**，根治思路（按侵入性递增）：

1. **把 src-cache key 加 OS 维度** —— `rusty-v8-src-v<version>-<os>`。
   跨 OS 不再共享源码 cache，也就不需要 Purge rust-toolchain，VERSION
   mtime 稳定，彻底消除触发源。代价：src-cache 数量翻倍（Linux +
   macOS 各一份，~360MB × 2）。**推荐**。
2. **restore cache 后对 VERSION 文件 `touch -r` 到已知固定 mtime**
   —— 骚操作，但不改 cache 结构。需要一个 sentinel mtime（比如仓库
   里某个固定文件）。
3. **禁用受影响 target 的 v8-deps cache**（`ios-device` 或 `ios-sim`
   专门 skip）—— 牺牲确定性换 CI 可预期性，warm cost 永远是 cold
   cost。不划算。
4. **在 `Purge OS-specific rust-toolchain` step 内只删 bin/ 子目录**
   保留 VERSION —— 小改，但需验证 V8 build.rs 在半残 rust-toolchain
   下的重建逻辑是否正确。

## 7. DIAG step 是否保留

**保留**。下次慢 case 自然发生时（sync 每日跑或新版本手动 build）
会自动捕获数据。DIAG 成本：每个 iOS job 多跑 ~2s。

DIAG 实际内容见 `build.yml` 里的 `DIAG pre-build snapshot (iOS)` 和
`DIAG post-build snapshot (iOS)` 两 step（`if: contains(matrix.target,
'apple-ios')` + cache-hit 门）。撤掉时直接删这两 step 即可。

## 8. 相关引用

- `docs/build-operations.md` §3.3 的"遗留迷惑案例"即本文档调查对象的
  早期观察。
- `docs/build-operations.md` §4.1 解释为何需要 Purge rust-toolchain
  （OS-绑定二进制不能跨 OS 共享），也就是慢 case 触发源的设计由来。
