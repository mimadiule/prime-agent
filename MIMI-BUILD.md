# mimi 构建指南(下次合并看这里)

> 个人 fork 自用构建。本文件 + CHANGELOG.md 的 `0.10.5-mimi.1` 条目 + README 顶部
> 徽标横幅,是每次重新合并时唯一需要维护的三处说明。

## 版本规则(重要)

- **Cargo/workspace 版本、tag、`--version` 输出全部保持官方样式(如 `0.10.5`)。**
  两个硬约束:`release.yml` 规定 tag 含 `-` 不真实编译;构建后有 "Livecheck gate"
  断言 `--version` 严格等于 tag 版本(本仓 token 无 workflow scope,改不了门禁)。
- fork 的区分标识在 **TUI 标题栏**:`chrome.rs` 显示 `v0.10.5-mimi.N`。
  升后缀版本只动一处(搜 `-mimi.`):
  `crates/pa-tui/src/chrome.rs` → `let version = format!("v{}-mimi.N", ...)`
  (更新检查、握手、build_id 保持官方版本号,便于和上游 release 比较,也避免
  版本协商误报。)

## 合并流程(0.10.5 实例,2026-11-19 执行)

1. 基座:官方 release tag(`git switch -C mimi-0.10.5 v0.10.5`)。
   0.10.3 起 tag 基座成为默认:官方发布节奏加快,open PR 在下个 tag 前多半会被
   收编或关闭,先查状态再动手,冲突率最低。tag-check 只比较 tag 与 Cargo.toml,
   官方 release commit 已把 Cargo.toml 升到同版本,天然通过。
2. 先查我们旧合入 PR 的上游状态(`gh pr view N --json state,mergedAt`):
   - MERGED 且 mergedAt < tag 时间 → 已随本 tag 收编,跳过;
   - MERGED 且 mergedAt > tag 时间 → 将随下个 tag 收编,跳过(如 0.10.5 轮的
     #3867:合并晚于发布 3 小时,仍要手工带);
   - OPEN → 重跑 squash 合入;
   - CLOSED → 用户决策:带(需在最新基座上重新验证)或不带。
3. 逐个尝试(`merge --squash` 冲突没有 MERGE_HEAD,弃用 `git reset --hard HEAD`):
   ```
   git fetch origin pull/N/head:pr-N   # 注意从上游(origin)拉,不是 fork
   git merge --squash pr-N || { git reset --hard HEAD; }
   git commit -m "PR #N: <subject>"
   ```
   ⚠️ CLOSED 的 PR 在官方重实现后冲突就不要硬上(无本地 Rust 无法编译验证):
   - #3847(内核驻留):0.10.3 轮起放弃——上游 v0.10.3 已用新架构
     (core_state.rs/BuiltHandles)自带 bash 级驻留门控,PR 的 asyncio 级扩展
     (`background_work_probe`)硬合会双机制打架。
   - #3792(customMessage 拒投):0.10.5 轮起放弃——0.10.3 时还能干净合入,
     但 v0.10.5 重写了注入路径(session_engine_impl.rs 加了脱敏安全日志)并把
     回合结算换成 `held_run_end` 新语义,3 个 daemon 文件各 1 处深层冲突,
     属架构分歧而非文本冲突。
4. mimi 专属改动每次都要重放:
   - ConPTY 中文修复:`git checkout <mimi.2-提交> -- crates/pa-types/src/platform/windows_console.rs`
     (上游持续没动这个文件,整文件取回即完整修复,含内置测试);
   - `chrome.rs` 标题栏标记(见版本规则);
   - 本文件 + CHANGELOG + README 横幅三处文档。
5. `git push fork HEAD:refs/heads/mimi-0.10.X` + `git push -f fork HEAD:refs/tags/v0.10.X`
   (tag 指向我们的合并提交而非官方 release commit,覆盖推)。

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
- 验证:解压后对二进制搜串(tar.gz 压缩流直接搜无效),`-mimi.` 比 `mimi.N` 稳
  (编译器串池分片);`--version` 应输出官方版本号。
- 2026-11-13 起 token 已带 workflow scope(设备码授权),推上游 workflow 变更、
  merge-upstream 同步 fork main 均可用。

## 历史合入记录

### 0.10.2-mimi.1(2026-11-02,基座 main)
成功(8):#3769 #3867 #3792 #3888 #3825 #3183 #3847 #3872
冲突放弃(13):#3201 #3311 #3829 #2530 #3615 #3354 #3589 #3328 #3863 #2451 #1146 #3411

### 0.10.3-mimi.1(2026-11-13,基座 v0.10.3 = d649b93cf)
- 已随 v0.10.3 收编:#3769 #3888 #3829;#3825 随 v0.10.4 出
- 合入(4):#3867 #3792 #3183 #3872;放弃:#3847(上游重实现)

### 0.10.5-mimi.1(2026-11-19,基座 v0.10.5 = 033eea170)
- 已随 v0.10.5 收编:#3825;#3867 官方 MERGED 但晚于打点 3 小时,仍手工带
- 合入(3):#3867 #3183 #3872
- 放弃:#3792(注入路径重写+held_run_end,见上);#3847(维持)
- mimi 重放:ConPTY 修复(79a468d78 整文件)+ chrome.rs `v0.10.5-mimi.1` + 三处文档
