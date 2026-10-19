# GitHub Pages 部署

MkDocs 将 Markdown 文档构建成静态网站，Material 提供主题与检索界面。GitHub Pages 托管构建后的页面。

## 本地检查

```bash
python -m pip install -e '.[docs]'
python -m mkdocs serve
python -m mkdocs build --strict
```

严格构建会把文档链接等警告当作错误。生成目录是 `site/`，无需提交到源码分支。

## 配置 GitHub

1. 将本项目上传到自己的仓库，保留 `.github/workflows/docs.yml`。
2. 进入仓库 Settings → Pages。
3. 在 Build and deployment 的 Source 中选择 GitHub Actions。
4. 推送到 `main` 或 `master`，或者在 Actions 中手动运行 Documentation。
5. 打开成功部署任务提供的页面地址。

如果默认分支不叫 `main` 或 `master`，修改 workflow 的 branches。工作流使用仓库自带的 GITHUB_TOKEN，不需要另外创建个人令牌。

`site_url` 没有伪造为任何现有仓库。自定义域名或分享网站前，可在 `mkdocs.yml` 设置你自己的实际 URL。文档内部使用相对路径，默认项目子路径可部署。

## 工作流结构

Build job 安装文档依赖、运行严格构建，并上传 `site/`。Deploy job 通过官方 Pages action 发布已构建产物。Deploy 拥有 `pages:write` 与 `id-token:write`，仓库内容权限为只读。

CI 工作流运行 Python 版本矩阵、CMake/SWIG 集成测试、独立 C++ CTest 和 Java 源码测试，以及 JPype 集成测试。

## 可选命令行发布

```bash
python -m mkdocs gh-deploy --strict
```

这是另一种基于 `gh-pages` 分支的部署方式。若使用它，应将 Pages Source 改为对应分支。不要同时混用两套发布配置。

官方说明：

- [MkDocs deployment](https://www.mkdocs.org/user-guide/deploying-your-docs/)
- [Material publishing](https://squidfunk.github.io/mkdocs-material/publishing-your-site/)
- [GitHub Pages custom workflows](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
