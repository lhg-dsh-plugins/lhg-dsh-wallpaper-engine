# 本版修复（BUGFIX）· 1.4.0 → 1.4.1

## 标题

`project.json` 的 `file` / `preview` 不做目录包含校验：第三方壁纸可让 `/preview`、`/media`、`/scene-files` 返回本机任意文件（上游 issue #252，安全类）。

## 现象与根因

扫描链把 `project.json` 里壁纸作者自填的 `file` / `preview` 直接 `resolve(dir, …)` 后登记进清单，解析结果从不检查是否仍在壁纸目录内：

- `lib/wallpaper-scan.js`（`enumerateWallpapersAsync`）：`proj.fileAbs = resolve(dir, proj.file)`、`proj.previewAbs = resolve(dir, proj.preview)`；
- `lib/upload-dir.js`（`enumerateUploadsP` 的工程目录分支）：同样的裸 `resolve(abs, …)`。

清单随后由 `lib/inventory.js` 按 `tokenFor(fileAbs)` 登记进媒体表，`lib/routes/media-bytes.js` 按 token 原样发文件；scene/web 类型时 `/scene-files` 的围栏根又是 `dirname(fileAbs)`（`lib/serve.js`），围栏根随之被搬到目录外。于是 `project.json` 写 `"preview": "/etc/passwd"` 或 `"file": "../../../.ssh/id_ed25519"` 的第三方壁纸，被扫描到之后即可借本插件的本机 HTTP 服务读出用户可读的任意文件。

## 证据链（修复前 → 修复后）

最小复现（临时目录造夹具：workshop 内容根里放一个 `file` 用 `../..` 穿越、`preview` 用绝对路径指向目录外 secret 的壁纸 + 一个正常壁纸对照；上传目录里再放一个同形态工程目录；分别走 `enumerateWallpapersAsync` 与 `enumerateUploadsP`）：

- **修复前**（exit 1）：恶意条目被原样登记——
  `A entry id=evil1 fileAbs=<ISO>/secret-a.txt previewAbs=<ISO>/secret-b.txt`（两项均逃出壁纸目录），
  `B entry id=up-dir-evilproj fileAbs=<ISO>/secret-b.txt previewAbs=<ISO>/secret-a.txt`（同上）；正常壁纸不受影响。`VERDICT=BUG`。
- **修复后**（exit 0）：两个恶意条目整条被剔除，正常壁纸 `good1` 的 `fileAbs` / `previewAbs` 与修复前逐字一致。`VERDICT=FIXED`。

另有仓库内守卫判据（见下「改动文件」中 `test/verify-wallpaper-scan.mjs` 的 H 段）：
正向判据 H1 在修复前形态下判红（变异体负对照 H2 即把判据回退成裸 `resolve` 的形态，H1 对它判红，证明判据非恒真）。

## 改动文件

核心修复（3 个文件）：

- `lib/fs-paths.js`：新增纯函数 `resolveWithinDir(dir, rel)` —— 词法包含判据（真源）：
  `resolve` 后走 `relative()` 检查，相对结果为空 / 为 `..` / 以 `../` 开头 / 为绝对路径（跨卷）一律返回 `null`。判据口径与本仓既有 `resolveUploadFile` 的 `relative()` 惯例一致。
- `lib/wallpaper-scan.js`：`enumerateWallpapersAsync` 改走注入的 `resolveWithinDir`：主文件逃出 ⇒ 整条剔除；预览逃出 ⇒ 只丢预览（`previewAbs = null`）。工厂新增注入项 `resolveWithinDir`（与 `extOf` 同为「真源在 `lib/fs-paths.js`、注入进扫描族」的既有模式）。
- `lib/upload-dir.js`：`enumerateUploadsP` 工程目录分支同样改走注入的 `resolveWithinDir`（主文件逃出按「主文件缺失」剔除；预览逃出置 `null` 后落既有候选图回退链）。

装配与守卫：

- `lib/kernel/host-kernel.js`：从 `lib/fs-paths.js` 引入 `resolveWithinDir` 并注入两个 kit。
- `test/verify-wallpaper-scan.mjs`：注入真源 `resolveWithinDir`；新增 H 段判据（H1 正向 + H2 变异体负对照）与判据函数 `projectPathsStayInside`。
- `docs/ROUTE-INDEX.md`：按守卫要求用 `node test/tools/host-route-index.mjs --write` 重新生成（H 段注释新增了 `/media`、`/preview` 各一处守卫提及，两行计数 18→19、12→13，其余不变）。

版本与落点（非缺陷修复，fork 契约项）：

- `package.json` 版本 1.4.0 → 1.4.1、`repository.url` 改指本版仓库；`package-lock.json` 与 7 个 `packages/*/package.json` 同步同号（上游 S3.2「主包与所有子包同号」不变量，只改 version 行）。
- `src/about-assets.js` 的 `ABOUT_REPO_URL`、`scripts/build-client.mjs` 中对应的构建标记、`test/verify-client.mjs` 的 Star 按钮断言：同步指向本版仓库（上游守卫 `verify-about` 要求 package.json 与客户端仓库地址一致）。

## 复验结果

- 最小复现：修复前 exit 1（双链路逃逸）→ 修复后 exit 0，正常壁纸输出不变。
- `test/verify-wallpaper-scan.mjs`：68/68 通过（基线 66，新增 H1/H2）。
- 全量守卫链（`npm run verify`，92 步，逐个按清单命令复跑）：**91 步通过**；`tsc --noEmit` 与改动文件 `eslint` 均通过。
- 唯一未过：第 26 步 `test/verify-glass-runtime-browser.mjs` —— 沙箱无 Chromium，其断言明示「缺少 Chromium 浏览器」；已用 `git stash` 在**未改动的基线**上复跑同一守卫，报完全相同的错，属环境限制、非本次回归。
- 链在第 26 步短路后，27–92 步改为逐个按清单命令补跑：除第 38 步 `verify-route-index` 初次因索引未重生成判红（按其守卫提示重生成后通过）外，其余全过；版本同号后又对包面相关守卫（package-files / package-publish / deployment / release-set / subpackage-manifests / contracts / about / client / client-sync 等）单独复跑，均通过。

## 边界（本版未动、如实标注）

- **符号链接逃逸**：本版是词法包含判据，不做 `realpath` 复核 —— 壁纸目录内的符号链接指向目录外时仍可跟随。上游 issue #252 把 realpath 复核列为建议项之一；本版先堵住无需任何本地配合、仅靠 `project.json` 字段即可利用的绝对路径 / `../..` 穿越这一主洞，符号链接逃逸（需要攻击者先在目录内放下链接）留作后续项。
- **`/scene-files` 子路径围栏**：其本身的子路径校验是既有机制（#149/#169），不在本条射程；本版修的是围栏根的来源（`fileAbs` 不再可逃出），根可信后既有围栏语义不变。
- issue #252 建议 2 提到 PR #223 的图集字段（`custom` / `single`）：在基线 `3b0103b` 中全仓检索无此字段的路径解析点（`readProjectP` 只透传 `file` / `preview`），无对象可改，未动。
