# effective-requirements-analysis

把《有效需求分析（第 2 版）》(徐锋，电子工业出版社) 的方法论，编码成一个**可执行的 skill**：当用户抛来一句"加个功能""支持导出""这个流程太乱"时，agent 不会急着写 PRD，而是先**向用户提问**，把方案级需求还原成问题级需求，再产出全套可追溯的需求文档。

> **需求 = 预期 − 现状。**
> 我们才是解决方案专家，客户只是问题专家。

---

## 这个 skill 解决什么

大多数 AI 写需求的方式是：用户说"加个导出按钮"→ agent 照做 → 生成一份看起来很完整、但每一条都是猜的 PRD。问题在于用户提的永远是**方案级需求**，而方案背后的问题、频率、现状、约束，全都还是未知。

这个 skill 用**阶段闸门**把这件事挡死：

| 阶段 | 允许产出 | 禁止产出 |
|---|---|---|
| **G0 尚未提问** | 《需求澄清问题清单》《访谈记录与假设登记册》 | 任何其他文件 |
| **G1 已提问、未收到答复** | 同上两份，可追加新问题（`Q-xx`） | 问题卡片、干系人档案、流程、场景、模型、功能清单、SRS —— **包括标着"分析底稿""初稿""供讨论"的** |
| **G2 已收到第一轮答复** | 分析底稿可写，每条须标 `A-xx` 或 `S-xx` | 未标注来源的结论 |
| **G3 澄清完成** | 全套产物 | —— |

闸门不是靠自觉，是靠 `scripts/check_artifacts.py` 查出来的。

---

## 安装

### 通用（Claude Code / OpenCode / AspireCode / DeepSeek Harness / 通用 agents）

```bash
git clone https://github.com/yangdavip/effective-requirements-analysis-skill.git \
  ~/.skills-manager/skills/effective-requirements-analysis
```

按你的 harness 把这个目录软链到对应的 skills 目录即可：

```bash
ln -s ~/.skills-manager/skills/effective-requirements-analysis \
  ~/.workbuddy/skills/effective-requirements-analysis      # WorkBuddy
ln -s ~/.skills-manager/skills/effective-requirements-analysis \
  ~/.config/opencode/skills/effective-requirements-analysis # opencode / AspireCode
ln -s ~/.skills-manager/skills/effective-requirements-analysis \
  ~/.dsh/skills/effective-requirements-analysis             # DeepSeek Harness
ln -s ~/.skills-manager/skills/effective-requirements-analysis \
  ~/.agents/skills/effective-requirements-analysis           # 通用 agents
```

### 直接复制也行

把整个目录放进任意 harness 的 skills 目录即可，脚本无外部依赖，只用 Python 标准库。

---

## 目录结构

```
effective-requirements-analysis/
├── SKILL.md                      # 方法论主体：铁律、阶段闸门、编号体系、17 项任务
├── references/
│   ├── interview-playbook.md     # 提问技巧库（表象原因决策提问法、360 度分析法…）
│   ├── task-playbook.md          # 17 项关键任务与剪裁条件
│   └── artifacts.md              # 全套产物模板（问题卡片、管控点、领域模型…）
├── templates/
│   ├── srs-outline.md            # 需求规格说明书骨架
│   └── daily-change.md           # 变更/优化型需求分析模板
└── scripts/
    └── check_artifacts.py        # 三阶段自检脚本
```

---

## 自检脚本

**每个阶段跑一次，用对应阶段的参数**（阶段不同，检查项不同）：

```bash
python3 scripts/check_artifacts.py <产物目录> --stage G1   # 闸门自检
python3 scripts/check_artifacts.py <产物目录> --stage G2   # 编号闭合 + 溯源 + S-xx 合规
python3 scripts/check_artifacts.py <产物目录> --stage G3   # 全部检查（加撞号与措辞一致性）
```

退出码：`0` = 通过，`1` = 发现问题，`2` = 用法/目录错误。

### G1：闸门自检（管内容，不只是管文件数量）

只允许《需求澄清问题清单》与《访谈记录与假设登记册》各一份，且这两份文件的内容必须干净。命中以下任一即报错：

1. 标题出现"业务流程/领域模型/功能需求/问题卡片"等分析产物名
2. 表头出现"流程/场景/模块/步骤/责任人"等分析结论特征词
3. 出现 `Q/A/S/T` 之外的前缀编号（`FP-01`/`US-01`/`D-01`…，定义或引用都算）
4. 正文出现"该系统应包含…"这类强断言
5. 代码块里画流程图（mermaid 等）
6. 出现无编号的清单式条目（功能/能力清单）

**换标题躲不掉。** 把"功能清单"改叫"附：本次要做的"，只要内容形态还是分析结论，一样会被抓出来。

### G2：编号闭合 + 溯源 + S-xx 合规

- **编号闭合** —— 被引用的编号必须有定义
- **溯源完整** —— `F-xx`/`NFR-xx`/`AC-xx`/`MR-xx` 必须带 `A-xx` 或 `S-xx`
- **S-xx 纯度** —— 假设只能是"是/否"型判断，不是一套结构
- **S-xx 登记表两列** —— 「验证方式」与「验证不成立的后果」必须存在且非空
- **前缀撞号** —— 同一前缀在两套不相干的族系里指代两件事 → 报错

> 溯源只对**需求条目**强制。`BR-xx` 业务规则、`US-xx` 业务场景、`PC-xx` 问题卡片等是**分析产物**，引用上游事实即可，不要求直达 `A-xx`。

### 撞号 vs 引用

同一编号在多份产物里出现**本身不是问题** —— SRS 摘录、优先级评估、变更分析表都会复述上游编号，那是引用。脚本按**语义角色**判族系：看这条记录**待在哪一栏**（表头列名 / 上级章节），不看它叫什么名字。

- 同一前缀在两套不相干的族系里指代两件事 → **报错**
- 同一编号在多份产物里出现但同族系 → 不报错；措辞不同 → **提示**

---

## 编号体系

| 前缀 | 含义 | 归属产物 |
|---|---|---|
| `Q-xx` | 待答复的问题 | 问题清单 |
| `A-xx` | 用户已确认的事实 | 访谈记录（**全项目唯一定义处**） |
| `S-xx` | 分析方假设（须写明验证方式与不成立后果） | 访谈记录（**全项目唯一定义处**） |
| `T-xx` | 业务术语定义 | 访谈记录 |
| `PC-xx` | 问题卡片 | 问题卡片 |
| `SD-xx` / `SP-xx` | 干系人 / 阻力点 | 干系人档案 |
| `CP-xx` | 管控点 | 管控点分析 |
| `FP-xx` | 业务流程 | 业务流程列表 |
| `US-xx` | 业务场景 | 业务场景描述 |
| `BR-xx` | 业务规则 | 业务规则描述 |
| `DC-xx` | 领域类 | 领域模型 |
| `QS-xx` | 质量场景 | 质量场景决策卡 |
| `MR-xx` | 维护场景 | 维护需求描述 |
| `C-xx` | 约束 | 约束描述 |
| `F-xx` | 功能需求 | 功能需求清单 |
| `NFR-xx` | 非功能需求 | 质量需求 |
| `AC-xx` | 验收标准 | 验收标准 |

一个前缀只指代一件事。已被占用就换（如 `RN-xx` 给阻力点），**不要复用**。

---

## 方法论要点

- **四条主线**：价值需求 / 功能需求 / 数据需求 / 非功能需求，外加业务规则与约束
- **17 项关键任务**（含剪裁条件）见 `references/task-playbook.md`
- **必答三问**：①提出者与使用者是否一致 ②是否涉及潜在影响者 ③是否能代表主流用户的声音
- **五类最小需求单元**、**四色建模法**、**场景—挑战—方案**
- **管控点 → 指标 → 报表**三层：报表清单不能直接从字段列得出，先问管控点（管理意图 Why）
- **价值评估**不搞多维加权平均 —— 一平均就把低频高价值需求埋没了

---

## 触发方式

在支持 skill 的 harness 里直接提需求即可，例如：

> 体检预约系统现在经常撞单，前台说在系统里给每个时段加个人数上限就行了，帮我把这个需求理清楚。

skill 会先反问 5–8 个有抓手的问题，产出《需求澄清问题清单》让你回答，**在你回答之前不会写任何 PRD**。

---

## 来源

方法论出自《有效需求分析（第 2 版）》，徐锋著，电子工业出版社出版。本仓库是对书中经验的可执行编码，仅用于个人工程实践。

## License

MIT
