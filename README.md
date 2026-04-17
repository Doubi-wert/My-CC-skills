# My CC Skills

**[NOTICE] This README is fully AI-generated. | 此 README 由 AI 完全生成。**

Personal Claude Code skills for iterative critique and paper learning.

---

# 我的 Claude Code Skills

个人 Claude Code 技能集合，包含批判性迭代和论文学习两个技能。

---

## Skills | 技能

### iterative-critique

[English](#iterative-critique-1) | [中文](#iterative-critique-中文)

A dual-agent critique-revision loop for generating high-quality documents and code.

- **Generator (G)**: Creates content based on reference documents
- **Critic (B)**: Reviews content against reference for inconsistencies
- **Iterative**: Continues until feedback drops below threshold

**Use cases**:
- Iteratively improve documents or code
- Critically review content against reference materials
- Multi-round revision based on academic/technical standards

> **WARNING | 警告**: Iterative critique involves multiple rounds of dual-agent review and revision. This process may consume a significant amount of tokens. Please be prepared for higher-than-usual API costs and ensure you have sufficient budget before initiating a critique session.

**适用场景**:

批判性迭代改进循环，使用双代理协同工作：

- **Generator (G)**: 基于参考文档生成内容
- **Critic (B)**: 从批判角度审查内容是否符合参考文档
- **迭代**: 持续改进直到反馈减少到阈值以下

**适用场景**:
- 迭代改进文档或代码
- 批判性审查内容是否符合参考文档
- 基于学术/技术标准进行多轮修订

> **警告**: 批判迭代涉及多轮双代理审查与修订，该过程可能消耗大量 token。请在使用前做好心理预期和预算准备。

---

### paper-learning

[English](#paper-learning-1) | [中文](#paper-learning-中文)

Generate teaching documents from academic papers using a two-layer critique system.

- **Chapter-level (Meso)**: Iterate on individual sections
- **Global-level (Macro)**: Cross-chapter consistency checks
- **Harmonizer (H)**: Coordinates version tracking and convergence

**Use cases**:
- Generate tutorials from research papers
- Create teaching materials with citation consistency
- Cross-reference validation across document sections

#### paper-learning (中文)

基于学术论文生成教学文档，采用双层批判迭代架构：

- **章节级 (Meso)**: 逐章迭代处理
- **整体级 (Macro)**: 跨章节一致性检查
- **Harmonizer (H)**: 协调版本追踪和收敛判定

**适用场景**:
- 基于论文生成教学指南
- 创建引用一致的教材
- 跨章节引用验证

---

## Quick Start | 快速开始

```markdown
# iterative-critique
"帮我写一个...并迭代改进"
"批判性地审查这份文档是否符合..."

# paper-learning
"基于这篇论文生成...教学文档"
"帮我学习这篇论文并生成笔记"
```

---

## Architecture | 架构

Both skills follow a similar dual-agent pattern:

| Agent | Role |
|-------|------|
| G (Generator) | Content generation |
| B (Critic) | Critical review |
| H (Harmonizer) | Coordination & convergence (paper-learning only) |

两个技能都遵循双代理模式：

| 代理 | 角色 |
|------|------|
| G (Generator) | 内容生成 |
| B (Critic) | 批判性审查 |
| H (Harmonizer) | 协调与收敛（仅 paper-learning） |

---

## File Structure | 文件结构

```
skills/
├── iterative-critique/
│   └── skill.md
├── paper-learning/
│   └── skill.md
└── README.md
```

---

## Contributing | 贡献

Contributions are welcome! Please feel free to submit issues or pull requests.
For detailed contribution guidelines, refer to the documentation.

欢迎贡献！请随时提交问题或拉取请求。详细贡献指南请参阅文档。

---

## License

MIT
