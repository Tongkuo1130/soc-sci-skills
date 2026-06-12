# soc-sci-skills

> Social Science Research Skills for AI Agents — 社会科学研究 AI 技能合集

为 WorkBuddy 设计的社会科学研究方法技能包，让 AI 智能体能够引导你完成严谨的学术分析流程。

---

## 已收录技能

| 技能 | 说明 | 依赖 |
|------|------|------|
| [tna-analysis](./tna-analysis/) | 转换网络分析（Transition Network Analysis）交互式向导。从环境检测到统计验证再到可复现报告，全流程引导。 | R + tna 包 |

---

## 安装方式

### 方式一：GitHub 直装（推荐）

在 WorkBuddy 对话中输入：

```
从 GitHub 安装 skill：https://github.com/Tongkuo1130/soc-sci-skills/tree/main/tna-analysis
```

### 方式二：手动复制

```bash
git clone https://github.com/Tongkuo1130/soc-sci-skills.git
cp -r soc-sci-skills/tna-analysis ~/.workbuddy/skills/
```

重启 WorkBuddy 后，说"TNA 分析"或"转换网络分析"即可触发。

---

## 设计原则

- **统计验证不可跳过** — 每个分析结论必须经过统计检验，不搞可视化即结论
- **环境适配优先** — R 没装就生成脚本 / 给 Shiny App 备选，绝不 AI 模拟结果
- **小白友好** — 你只需要数据，剩下的技能一步步带你走
- **可复现** — 每次分析输出完整 `.R` 脚本 + `sessionInfo()`

---

## 贡献

欢迎 PR。新增技能请遵循如下目录结构：

```
skill-name/
├── SKILL.md          # 技能主文件
└── references/       # 参考文档（可选）
```
