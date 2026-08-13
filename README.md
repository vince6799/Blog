# 网络体验志

一个可直接部署到 GitHub Pages 的 Jekyll 博客。

## 发布

1. 在 GitHub 新建公开仓库。
2. 将本目录内的全部文件上传到仓库根目录。
3. 打开 `Settings → Pages`。
4. 选择 `Deploy from a branch`、`main` 和 `/(root)`，保存。

如果仓库名不是 `用户名.github.io`，请把 `_config.yml` 中的 `baseurl` 改为 `/仓库名`。

## 新增文章

复制 `_posts/2026-08-13-boostnet-review.md`，文件名改为：

`年-月-日-英文标题.md`

然后修改文件顶部的标题、摘要、分类和正文。提交后首页会自动出现新文章。
