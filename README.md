# Tech Notes

一个利用 Claude AI 辅助学习并记录的技术笔记博客，涵盖 Linux 系统编程、分布式系统、性能优化等领域。

## 关于本项目

笔记内容通过与 Claude 的深度对话整理而成——提问、探讨、验证，再沉淀为结构化的文档。Claude 在这里充当学习伙伴：帮助梳理知识体系、补充细节、指出盲点。

## 技术栈

- [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) — 文档站点框架
- GitHub Actions — 自动构建并部署到 GitHub Pages

## 本地运行

```bash
pip install -r requirements.txt
mkdocs serve
```

访问 `http://localhost:8000` 预览站点。

## 内容结构

```
docs/
└── linux/
    └── file_io.md    # 文件 I/O 系统性总结
```
