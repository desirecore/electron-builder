# DesireCore 维护的 electron-updater

> DesireCore's fork of `electron-updater`, carrying fixes for differential (block-map) downloads.
> Upstream: [electron-userland/electron-builder](https://github.com/electron-userland/electron-builder) `packages/electron-updater`.

electron-updater 没有独立仓库，源码在 electron-builder 的 monorepo 里，所以 fork 的是整个
electron-builder；我们只改、只发布 `packages/electron-updater`。

| 项 | 值 |
| --- | --- |
| 维护分支 | `desirecore/electron-updater-6.8.x` |
| 上游基线 | tag `electron-updater@6.8.9`（`bed3a9c4`） |
| 版本号 | `<上游版本>-desirecore.<n>`，如 `6.8.9-desirecore.1` |
| 发布物 | 本仓库 GitHub Release 附件 `electron-updater-<版本>.tgz` |
| Release tag | `electron-updater-v<版本>`，如 `electron-updater-v6.8.9-desirecore.1` |

## 为什么 fork：差量升级在我们的发布源上基本从不成功

2026-09 真机排查（10.0.150 → 10.0.155）：electron-updater 报
`Cannot download differentially, fallback to full download: Error: Response ends without calling any handlers`，
白等 43 秒后下载整包 543 MB。查下来是三个互相独立的缺陷，上游 6.8.9（最新正式版）与 7.0.0-alpha.7 都没修：

1. **只认 CRLF 的 multipart**（`DataSplitter`）。七牛 CDN（openresty `web cache`）的
   `multipart/byteranges` 响应用裸 LF 换行，不是 RFC 2046 要求的 CRLF。`DataSplitter` 只找
   `\r\n\r\n`，于是一段都切不出来，还把整个响应（一批可达 172 MB）攒进内存。
   **只要走七牛，多段差量必失败。**
2. **跨数据块的段头分隔符找不到**（`DataSplitter`）。段头没收全时，旧实现只在新数据块里找
   `\r\n\r\n`，之前攒下的缓冲从不参与查找；分隔符恰好被数据块边界切开就漏掉，错位到下一段。
   每批几百段时有可观概率撞上，结果是 sha512 校验失败、回落整包。
3. **批次成功后看门狗不撤**（`multipleRangeDownloader`）。任务超过 1000 个就分批请求；每批响应
   `end` 后无条件起一个 10 秒定时器 `reject`，批次成功也不清。于是第 1 批结束 10 秒后，只要后续批次
   还没下完，整个差量就被判失败——**格式规范的源（我们的 nginx 一级源）也救不了大版本跨度。**

修法见本分支提交；回归测试在 `test/src/updater/differentialMultipartTest.ts`
（CRLF / 裸 LF × 各种分块粒度，以及「第 1 批完成后拨快 11 秒，第 2 批不得被判失败」）。
这些测试在上游原版上全部失败，在本分支上全部通过。

## 构建与发布

```bash
pnpm install --frozen-lockfile --ignore-scripts
pnpm exec tsc --build packages/builder-util-runtime packages/electron-updater
TEST_FILES=differentialMultipartTest,downloadPlanBuilderTest pnpm ci:test

# 版本号写进 packages/electron-updater/package.json 后
cd packages/electron-updater && pnpm pack --pack-destination ../..
# workspace:* 会被 pnpm 解析成实际版本（6.8.9 基线下 builder-util-runtime = 9.7.0）

gh release create electron-updater-v6.8.9-desirecore.1 electron-updater-6.8.9-desirecore.1.tgz \
  --repo desirecore/electron-builder --target desirecore/electron-updater-6.8.x
```

发布前与 npm 官方同版本 tarball 逐文件比对：除改过的文件与 `package.json` 版本号外应逐字节相同。

## DesireCore 如何引用

`package.json` 直接写 Release 附件 URL：

```json
"electron-updater": "https://github.com/desirecore/electron-builder/releases/download/electron-updater-v6.8.9-desirecore.1/electron-updater-6.8.9-desirecore.1.tgz"
```

**URL 形状有硬约束**：electron-builder 打包前会 `semver.coerce()` 这个依赖字符串并要求 `>=4.0.0`，
不满足直接抛 `InvalidConfigurationError`。coerce 取的是字符串里**第一个**数字序列，所以 URL 里
版本号之前不能出现任何数字（这就是 tag 用 `electron-updater-v<版本>` 而不是 `…@<版本>` 的原因：
`@` 会被编码成 `%40`，coerce 会读出 `406.8.9`）。`file:` 形式也不行——它会去该路径下读
`package.json`，且相对路径按 app-builder-lib 自己的目录解析。

## 跟进上游新版本

1. `git fetch upstream 'refs/tags/electron-updater@<新版本>:refs/tags/electron-updater@<新版本>'`
2. 从新 tag 拉 `desirecore/electron-updater-<新 minor>.x`，`git cherry-pick` 本分支上的修复提交
3. 先跑一遍测试确认上游是否已自行修复（若修复，对应补丁可以丢弃）
4. 版本号改成 `<新版本>-desirecore.1`，构建、比对、发 Release，再更新 DesireCore 的依赖 URL 与 lock

这三处修复已提给上游：**[electron-userland/electron-builder#10192](https://github.com/electron-userland/electron-builder/pull/10192)**（基于 upstream/master 重做，带 changeset 与同一份回归测试）。上游合并并发版后，DesireCore 应切回 npm 官方包，本 fork 随之退役。
