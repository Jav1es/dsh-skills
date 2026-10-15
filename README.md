# dsh-skills

可复用的 DSH / Agent Skills 集合。

## Skills

| Skill | 用途 |
|---|---|
| [`knowledge-guide`](knowledge-guide/SKILL.md) | 知识检索与回答规程：先检索再作答、强制标来源与置信度、未命中先做正对照 |
| [`resume-jd-tailor`](resume-jd-tailor/SKILL.md) | 简历 JD 适配：8 项体检 → 逐段映射 → 成果式改写 → 诚实口径检查 |
| [`static-portfolio-build`](static-portfolio-build/SKILL.md) | 静态多页作品集：数据驱动架构、SVG 示意图、3D 与动效、性能优化、部署验收 |

## 使用

每个 skill 是独立目录，内含 `SKILL.md`（带 YAML frontmatter：`name` / `description` / `metadata`）。

放到 skill 目录即可被识别，例如：

```bash
git clone https://github.com/Jav1es/dsh-skills.git
cp -r dsh-skills/* ~/.dsh/skills/
```

## 说明

- `knowledge-guide` 与 [`dsh-knowledge-guide`](https://github.com/Jav1es/dsh-knowledge-guide) 内容同源，本仓库是其多 skill 集合版。
- `static-portfolio-build` 里的坑位记录来自 [`portfolio`](https://github.com/Jav1es/portfolio) 项目的实际迭代。

## License

MIT
