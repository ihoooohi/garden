# ZY's Garden 🌱

> 应灼灿的个人知识库 / 数字花园

🌐 **在线访问**：https://ihoooohi.github.io/garden/

## 这是什么

一个慢慢长大的数字花园 —— 用来沉淀读过的好文章、写过的代码、想过的事情。

用 [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) 搭建，部署在 GitHub Pages 上。

## 本地预览

```bash
pip install mkdocs-material
mkdocs serve
# 然后打开 http://localhost:8000
```

## 添加内容

1. 在 `docs/<分类>/` 下新建 `.md` 文件
2. 在 `mkdocs.yml` 的 `nav` 里加一行
3. `git push` —— GitHub Actions 自动部署

## 目录

```
docs/
├── index.md            # 首页
├── reading/            # 📚 读文沉淀
├── tech/               # 🛠️ 技术笔记
├── thoughts/           # 💭 思考与随笔
└── tags.md             # 🏷️ 标签索引
```

---

Source: [github.com/ihoooohi/garden](https://github.com/ihoooohi/garden)
