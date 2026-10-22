# UPSTREAM

本仓库是上游项目的修复分支（fork），出处与基线如下：

| 项 | 值 |
|---|---|
| 上游仓库 | https://github.com/elysia395/dsh-wallpaper-engine |
| 上游 npm 包名 | `dsh-plugin-wallpaper-engine` |
| 基线版本 | 1.4.0 |
| 基线 commit（固定 SHA） | `3b0103bc1d5431840208966ce67155f392c4095c` |
| 拉取日期 | 2026-10-23 |
| 原始作者 | elysia395（见 LICENSE：Copyright (c) 2026 elysia395） |
| 许可证 | MIT（LICENSE 文件原样保留，一字未改） |
| 本版仓库 | https://github.com/lhg-dsh-plugins/lhg-dsh-wallpaper-engine |
| 本版版本 | 1.4.1（主包与全部子包同号，遵循上游 S3.2 同号不变量） |

本版相对基线的改动只有两类：

1. **缺陷修复**：上游 issue #252（`project.json` 的 `file` / `preview` 不做目录包含校验），
   证据链与改动清单见 [BUGFIX.md](BUGFIX.md)。
2. **落点仓一致性改名**：`package.json` 的 `repository.url`、客户端关于页的仓库地址
   （`src/about-assets.js` 的 `ABOUT_REPO_URL` 及其构建标记、对应守卫断言）指向本版仓库——
   这是上游守卫 `verify-about` 的硬性不变量（package.json 与客户端必须指向同一仓库），
   不改则守卫判红。上游文档正文、作者署名与 LICENSE 均未改动。


---

## 修复版版权

本修复版的修改部分 © 2026 lhg-dsh-plugins（刘洪光），修改内容见 [BUGFIX.md](./BUGFIX.md)；原始代码版权归原作者所有，见仓库内 LICENSE 与上方出处。
