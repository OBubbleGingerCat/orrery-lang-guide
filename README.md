<p align="center">
  <img src="docs/public/logo.svg" width="120" alt="Orrery logo">
</p>

# Orrery guide · Orrery 指南

The website of Orrery, for writing cycle-level models of whole systems in
Rust: <https://obubblegingercat.github.io/orrery-lang-guide/> (English,
with a Chinese version at `/zh/`).

Orrery 的网站，讲怎样用 Rust 给整个系统写周期级模型：<https://obubblegingercat.github.io/orrery-lang-guide/zh/>。

## Working on the site · 修改网站

The site is built with [VitePress](https://vitepress.dev). The English pages
are in `docs/`, the Chinese ones in `docs/zh/` with the same names; the
navigation and sidebars are in `docs/.vitepress/config.mts`.

网站用 VitePress 生成。英文页面在 `docs/`，中文页面在 `docs/zh/`，文件名一一对应；
导航和侧栏在 `docs/.vitepress/config.mts`。

```sh
npm ci            # install
npm run dev       # live preview at http://localhost:5173/orrery-lang-guide/
npm run build     # build into docs/.vitepress/dist
```

Every push to `main` builds the site and publishes it on GitHub Pages
(`.github/workflows/deploy.yml`).

每次推到 `main` 都会自动构建并发布到 GitHub Pages。

## License · 许可

Code samples in this repository, both the code blocks in the documentation
and any source files, are licensed under the
[MIT No Attribution License](LICENSE-CODE) (MIT-0). Copy them into your own
models freely; no attribution is needed.

All other content is licensed under the
[Creative Commons Attribution 4.0 International License](LICENSE) (CC BY 4.0).

本仓库里的示例代码，包括文档里的代码块和任何源文件，以
[MIT No Attribution License](LICENSE-CODE)（MIT-0）许可发布，可以随意抄进自己的模型，不需要署名。

其余内容以
[Creative Commons Attribution 4.0 International License](LICENSE)（CC BY 4.0）许可发布。
