**English** | [简体中文](https://github.com/Doubi-wert/My-CC-skills/blob/main/readme_zh.md)

# My CC Skills

**[NOTICE] This README is fully AI-generated.**

A collection of Claude Code skills for iterative critique and paper learning.

---

## Skills

### iterative-critique

A dual-agent critique-revision loop for generating high-quality documents and code.

- **Generator (G)**: Generates content based on reference documents
- **Critic (C)**: Reviews content against reference for inconsistencies
- **Iterative**: Continues until feedback drops below threshold

**Use cases**:
- Iteratively improve documents or code
- Critically review content against reference materials
- Multi-round revision based on academic/technical standards

> **WARNING**: Iterative critique involves multiple rounds of dual-agent review. This may consume significant tokens. Ensure sufficient budget before initiating.

---

### paper-learning

Generates teaching documents from academic papers using a two-layer critique system. Internally invokes **iterative-critique** for content review and refinement.

- **Chapter-level (Meso)**: Iterate on individual sections
- **Global-level (Macro)**: Cross-chapter consistency checks

**Use cases**:
- Generate tutorials from research papers
- Create teaching materials with citation consistency
- Cross-reference validation across document sections

> **INPUT NOTE**: Requires paper content in `.md` format. Use OCR or copy tools for extraction — but for papers with heavy math or images, OCR often causes errors. **Manual review recommended.** **Do not input PDF directly; convert to `.md` first.**

---

## Quick Start

```markdown
# iterative-critique
"Help me write...with iterative improvement"
"Critically review this document against..."

# paper-learning
"Generate a tutorial document from this paper"
"Help me learn this paper and create notes"
```

---

## Architecture

Both skills use a dual-agent pattern:

| Agent | Role |
|-------|------|
| G (Generator) | Content generation |
| C (Critic) | Critical review |

---

## File Structure

```
skills/
├── iterative-critique/
│   └── skill.md
├── paper-learning/
│   └── skill.md
├── README.md
└── readme_zh.md
```

---

## Contributing

Contributions welcome! Please submit issues or pull requests.

---

## License

MIT
