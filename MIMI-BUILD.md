# mimi 构建指南(下次合并看这里)

> 个人 fork 自用构建。本文件 + CHANGELOG.md 的 `0.10.3-mimi.1` 条目 + README 顶部
> 徽标横幅,是每次重新合并时唯一需要维护的三处说明。

## 版本规则(重要)

- **Cargo/workspace 版本、tag、`--version` 输出全部保持官方样式(如 `0.10.3`)。**
  两个硬约束:`release.yml` 规定 tag 含 `-` 不真实编译;构建后有 "Livecheck gate"
  断言 `--version` 严格等于 tag 版本(本仓 token 无 workflow scope,改不了门禁)。
- fork 的区分标识在 **TUI 标题栏**:`chrome.rs` 显示 `v0.10.3-mimi.N`。
  升后缀版本只动一处(搜 `-mimi.`):
  `crates/pa-tui/src/chrome.rs` → `let version = format!("v{}-mimi.N", ...)`
  (更新检查、握手、build_id 保持官方版本号,便于和上游 release 比较,也避免
  版本协商误报。)

## 合并流程(0.10.3 实例,2026-11-13 执行)

1. 基座:官方 release tag(`git switch -C mimi-0.10.3 v0.10.3`)。
   0.10.3 起 tag 基座成为默认:官方发布节奏加快,open PR 在下个 tag 前多半会被
   收编或关闭,先查状态再动手,冲突率最低。tag-check 只比较 tag 与 Cargo.toml,
   官方 release commit 已把 Cargo.toml 升到同版本,天然通过。
2. 先查我们旧合入 PR 的上游状态(`gh pr view N --json state,mergedAt`):
   - MERGED 且 mergedAt < tag 时间 → 已随本 tag 收编,跳过;
   - MERGED 且 mergedAt > tag 时间 → 将随下个 tag 收编,跳过;
   - OPEN → 重跑 squash 合入;
   - CLOSED → 用户决策:带(需在最新基座上重新验证)或不带。
3. 逐个尝试:
   ```
   git fetch origin pull/N/head:pr-N   # 注意从上游(origin)拉,不是 fork
   git merge --squash pr-N || { git merge --abort; }
   git commit -m "merge PR #N: <subject>"
   ```
   ⚠️ CLOSED 的 PR 若在官方重实现后被放弃(如 #3847),冲突解不开也不要硬上:
   上游 v0.10.3 已用新架构(core_state.rs/BuiltHandles)自带 bash 级驻留门控,
   #3847 的 asyncio 级扩展背景工作探针无法在无本地 Rust 的条件下安全移植。
4. mimi 专属改动每次都要重放:
   - ConPTY 中文修复:`git checkout <mimi.2-提交> -- crates/pa-types/src/platform/windows_console.rs`
     (上游没动这个文件,整文件取回即完整修复,含内置测试);
   - `chrome.rs` 标题栏标记(见版本规则);
   - 本文件 + CHANGELOG + README 横幅三处文档。
5. `git push fork HEAD:refs/heads/mimi-0.10.X` + `git push fork HEAD:refs/tags/v0.10.X`。

## 构建三坑(每次都会踩)

| 坑 | 解法 |
|---|---|
| fork 的 Actions 没真正启用,tag 推了没 run | `gh api -X PUT repos/mimadiule/prime-agent/actions/permissions -F enabled=true -F allowed_actions=all`(`-F` 不是 `-f`,布尔类型!),然后删 tag 重推触发 |
| `gh run watch` 偶发提前退出 | 用轮询脚本(45s 间隔),等 status=completed |
| SHA256SUMS 是 CRLF,`sha256sum -c` 报错 | `tr -d '\r'` 后手动比对 |

- 编译作业全绿、只有 `promotion gate -> release attach` 失败 = **正常**(fork 无
  官方 R2 密钥,publish 步骤 `test -n "$R2_ENDPOINT_URL"` 必挂),artifact 在失败前
  已上传,直接下载;GitHub Release 需手动挂(见 handoff.md 第 6 步)。
- 产物:`gh api repos/mimadiule/prime-agent/actions/runs/<run_id>/artifacts` 找
  `artifacts-<target>`,zip 里是 tar.gz + SHA256SUMS + manifest.json。
- 验证:二进制字节里搜 `mimi.N` 串(编译器可能把 `v0.10.3-mimi.1` 拆成串池分片,
  搜 `-mimi.` 更稳);`--version` 应输出 `0.10.3`。

## 0.10.2-mimi.1 实际合入记录(2026-11-02)

成功(8):#3769 #3867 #3792 #3888 #3825 #3183 #3847 #3872
冲突放弃(13):#3201 #3311 #3829 #2530 #3615 #3354 #3589 #3328 #3863 #2451 #1146 #3411
(均为堆叠分支或 20+ 文件大改;其中 #3829 官方已于 v0.10.3 自行收编)

## 0.10.3-mimi.1 实际合入记录(2026-11-13)

基座:v0.10.3(release commit d649b93cf)。旧 8 PR 的上游状态:
- 已随 v0.10.3 收编,跳过:#3769 #3888 #3829(当初冲突放弃的,官方自己合了)
- #3825:官方 MERGED(晚于 v0.10.3 打点)→ 随 v0.10.4 出,跳过
- 合入成功(4):#3867 #3792 #3183 #3872
- **放弃:#3847(内核驻留)** —— 上游已用新架构重实现 bash 级驻留门控
  (core_state.rs `background_bash_probe` + `can_passivate_worker`,测试同名),
  PR 的 asyncio 级扩展(`background_work_probe`)无法无编译验证地移植,硬合会
  双机制打架。等上游(或 PR 作者重开)在 0.10.4 轮再看。

mimi 专属(本次重放):ConPTY UTF-8 代码页修复(windows_console.rs 整文件回取,
上游未动)+ chrome.rs `v0.10.3-mimi.1` + 三处文档。
