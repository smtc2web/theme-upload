# theme-upload

[smtc2web 主题商店](https://themes.smtc2web.org) 的自动发布流水线。主题仓库接入后，推送 `v*` tag 即可自动打包并发布到商店，无需在仓库中配置任何 secret。

## 快速开始

1. 在主题仓库新建 `.github/workflows/publish.yml`（内容见 [`examples/publish.yml`](./examples/publish.yml)）：

   ```yaml
   name: Publish theme
   on:
     push:
       tags: ["v*"]
     workflow_dispatch:

   jobs:
     publish:
       permissions:
         contents: read
         id-token: write
       uses: smtc2web/theme-upload/.github/workflows/publish.yml@v1
   ```

2. 确保仓库根目录有 `theme.toml`，且包含 `[smtc2web.theme]` 配置节：

   ```toml
   [smtc2web.theme]
   name = "my-theme"
   version = "0.1.0"
   author = "your-name"
   description = "主题简介"
   repository = "https://github.com/you/my-theme.git"
   tags = ["dark", "minimal"]
   screenshot = "screenshot.png"
   ```

3. 更新 `theme.toml` 的 `version`，打 tag 并推送：

   ```bash
   git tag v0.1.0
   git push origin v0.1.0
   ```

流水线会打包仓库内容（自动排除 `.git`、`.github`、`node_modules`，并套一层与仓库同名的根文件夹）、校验 `theme.toml`，然后通过 OIDC 发布到商店。

## 输入参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `store-url` | `https://themes.smtc2web.org` | 商店地址 |
| `folder` | 仓库名 | ZIP 内的顶层文件夹名 |
| `source-dir` | `.` | 打包的源目录，需包含 `theme.toml` |
| `build-command` | 空 | 打包前在 `source-dir` 中执行的构建命令 |
| `strict-version` | `true` | 打 tag 时校验 tag（去掉 `v`）与 `theme.toml` 的 `version` 一致 |

需要构建的主题（例如源码经打包后输出到 `dist`）可以这样调用：

```yaml
uses: smtc2web/theme-upload/.github/workflows/publish.yml@v1
with:
  build-command: pnpm install && pnpm build
  source-dir: dist
```

## 认证原理

工作流使用 GitHub Actions 的 [OIDC](https://docs.github.com/en/actions/reference/security/oidc) 获取短期身份令牌，商店校验签名、受众（`smtc2web-themes`）以及 `job_workflow_ref` 必须指向本仓库的官方工作流。因此：

- 主题仓库不需要保存任何 token / secret；
- 只有通过本工作流发起的发布才会被接受；
- 商店按仓库归属绑定主题，其他仓库无法覆盖同名主题。

## 常见错误

| 错误 | 原因与处理 |
| --- | --- |
| 无法获取 OIDC Token | 调用工作流缺少 `permissions: id-token: write` |
| tag 与 theme.toml 中的 version 不一致 | 更新 `theme.toml` 的 `version` 后重新打 tag，或设置 `strict-version: false` |
| 该流水线未被授权向主题商店发布主题 | 必须通过 `smtc2web/theme-upload` 的官方工作流调用 |
| 主题标识 xxx 已被其他作者占用 | 主题由其他仓库首次发布；请换用别的标识或以作者身份在商店处理 |
| 版本 x.y.z 已发布 | 递增 `theme.toml` 的 `version` |
| 主题包必须包含且仅包含一个根文件夹 | 不要在打包目录中混入其他顶层文件，交由本工作流打包即可 |
