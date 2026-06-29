# Rein 文档与官网

[Rein](https://github.com/MaLunan/rein) 框架的文档与官网源。

- `docs/` —— 内部设计文档(DESIGN / code-guide / 开发计划 develop)
- `website/` —— 官网与文档站(MkDocs Material)

## 本地预览

```bash
pip install mkdocs-material
cd website && mkdocs serve   # → http://127.0.0.1:8000
```

## 部署

push 到 main 后,GitHub Actions(`.github/workflows/docs.yml`)自动部署到 GitHub Pages。
