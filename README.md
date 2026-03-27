# Hexo Reimu Blog

基于 `Hexo 8.1.1` 和 `hexo-theme-reimu 1.12.2` 的博客仓库，使用 GitHub Pages 自动部署。

## 本地运行

```bash
npm install
npm run server
```

默认访问地址：

```text
http://localhost:4000/blog/
```

如果你希望清理缓存后再启动：

```bash
npm run clean
npm run server
```

## 构建静态文件

```bash
npm run build
```

生成结果位于 `public/`。

## GitHub Pages 部署

1. 仓库名使用 `blog`
2. 推送到 `main` 分支
3. GitHub 仓库中进入 `Settings -> Pages`
4. `Build and deployment` 选择 `GitHub Actions`
5. 推送后等待 `.github/workflows/deploy-pages.yml` 完成

部署地址：

```text
https://celestialice.github.io/blog/
```

## Reimu 主题配置

- Hexo 主配置在 `_config.yml`
- Reimu 覆盖配置在 `_config.reimu.yml`
- 头像文件放在 `source/_data/avatar/avatar.webp`
- 关于页在 `source/about/index.md`
- 友链页在 `source/friend/index.md`
