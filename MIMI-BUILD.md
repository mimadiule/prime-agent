# mimi 构建指南(下次合并看这里)

> 个人 fork 自用构建。本文件 + CHANGELOG.md 的 `0.10.2-mimi.1` 条目 + README 顶部
> 徽标横幅,是每次重新合并时唯一需要维护的三处说明。

## 版本规则(重要)

- **Cargo/workspace 版本、tag、`--version` 输出全部保持官方样式 `0.10.2`。**
  两个硬约束:`release.yml` 规定 tag 含 `-` 不真实编译;构建后有 "Livecheck gate"
  断言 `--version` 严格等于 tag 版本(本仓 token 无 workflow scope,改不了门禁)。
- fork 的区分标识在 **TUI 标题栏**:`chrome.rs` 显示 `v0.10.2-mimi.1`。
  改后缀版本只动一处(搜 `-mimi.1`):
  `crates/pa-tui/src/chrome.rs` → `let version = format!("v{}-mimi.1", ...)`
  (更新检查、握手、build_id 保持官方版本号,便于和上游 release 比较,也避免
  版本协商误报。)

## 合并流程(以下次 0.10.3 为例)

1. `git clone --filter=blob:none https://github.com/PrimeIntellect-ai/prime-agent pa-build`
   (或已有仓库 `git fetch origin main` 后 `git switch -C mimi-0.10.3 origin/main`)。
   基座选 main 而非 release tag:PR 都基于 main,冲突率低一个量级,且官方已合并的
   修复(如 #3755 粘贴修复)天然自带。tag-check 只比较 tag 与 Cargo.toml,通过。
2. 候选筛选:全量抓 open PR(`gh api repos/PrimeIntellect-ai/prime-agent/pulls`,
   >2MB 输出落盘再解析),用 zev skill 两轮打分(价值分 + 自包含性),重叠图排除
   大型耦合 PR。
3. 逐个尝试(顺序:分值高、体积小、自包含优先):
   ```
   git fetch origin pull/N/head:refs/remotes/pr/N
   git merge --squash refs/remotes/pr/N || { git reset --hard HEAD; git clean -qfd; }
   # 成功则 git commit -m "PR #N: <subject>"
   ```
   堆叠 PR(基于其他未合并 PR 开发)必然冲突,直接放弃,不硬解。
4. `git push fork HEAD:refs/heads/mimi-0.10.X` + `git push -f fork HEAD:refs/tags/v0.10.X`。

## 构建三坑(每次都会踩)

| 坑 | 解法 |
|---|---|
| fork 的 Actions 没真正启用,tag 推了没 run | `gh api -X PUT repos/mimadiule/prime-agent/actions/permissions -F enabled=true -F allowed_actions=all`(`-F` 不是 `-f`,布尔类型!),然后删 tag 重推触发 |
| `gh run watch` 偶发提前退出 | 用轮询脚本(45s 间隔,见 watch2.sh 思路),等 status=completed |
| SHA256SUMS 是 CRLF,`sha256sum -c` 报错 | `tr -d '\r'` 后手动比对 |

- 编译作业全绿、只有 `promotion gate -> release attach` 失败 = **正常**(fork 无权限
  attach 到官方 release),artifact 在失败前已上传,直接下载。
- 产物:`gh api repos/mimadiule/prime-agent/actions/runs/<run_id>/artifacts` 找
  `artifacts-x86_64-pc-windows-msvc`,zip 里是 tar.gz + SHA256SUMS + manifest.json,
  exe 名为 `prime-agent.exe`,本地改名加 `-mimi.1` 区分。
- 验证:`./prime-agent.exe --version` 应输出 `0.10.2-mimi.1`。

## 0.10.2-mimi.1 实际合入记录(2026-11-02)

成功(8):#3769 #3867 #3792 #3888 #3825 #3183 #3847 #3872
冲突放弃(13):#3201 #3311 #3829 #2530 #3615 #3354 #3589 #3328 #3863 #2451 #1146 #3411
(均为堆叠分支或 20+ 文件大改;若其依赖的上游 PR 后续合并,重跑第 3 步即可)
