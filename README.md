# jd-double-column-card

**京东站内双列卡片·体验观察 Skill（Claude Agent Skill）**

面向 AI agent 执行的"双列卡片体验观察工具"：输入页面截图 + business_line，输出差异观察报告 + 体验设计视角评价。**不做纠错判决，做观察与讨论的抓手**。

---

## 快速上手（给 AI agent）

1. **克隆到 Claude Skills 目录**：
   ```bash
   git clone <this-repo> ~/.claude/skills/jd-double-column-card
   ```

2. **从入口文件开始读**：
   - `SKILL.md` — frontmatter 里的 `entry_for_agents` 指向主执行契约
   - 主执行契约：`prompts/page-level-review.md`

3. **标准调用格式**：
   ```yaml
   business_line: <推荐 | 点评 | 圈子 | 新品 | 个人中心 | 直播>  # 必填，人工声明
   page_name: <string>            # 必填
   page_screenshot: <path>        # 必填
   version: After                 # 可选
   focus: <string>                # 可选
   ```

4. **Skill 会按 Step 0-6 流程执行**，输出三段式/六段式观察报告。

---

## 核心设计

### 五大家族体系

| 家族 | 底部 DNA | 卡片主语 |
|---|---|---|
| 内容 | 头像 + 昵称 + 互动 | 创作者 |
| 商品 | 店铺 logo + 店铺名 + 销售数据 | 商家 |
| 圈子 | 圈子图标 + 圈子名 | 社群 |
| 点评 | 头像 + 昵称 + 互动 + 标题下种草数 | 推荐者 |
| 新品 | 按子类型变化 | 抹除主语（事件驱动） |

**家族身份由底部不变量锚定，不由封面决定**（L1-5 核心规约）。

### 规则分层

- **L1**：9 条跨家族规约（所有卡必须走）
- **L2**：家族专属细则（按 `card_type` 分支）

### 8 条核心原则（摘要）

1. 家族识别先于规则比对
2. 偏离即差异观察（不是违规判决）
3. 跨场景同物种识别（变体评价一票否决标准）
4. 创新的举证责任倒置
5. 流级信息类型一致性
6. 识别深度匹配规范深度
7. **结构优先、内容无涉**（只看设计师可控的结构，不看用户上传的内容）
8. **动态数据不作家族识别硬条件**（种草数/观看量是运行时数据，不参与家族判定）

---

## 目录结构

```
jd-double-column-card/
├── SKILL.md                         # 入口，含 frontmatter + 架构 + 8 条原则
├── README.md                        # 本文件
├── references/
│   ├── business-baselines.md        # ★ 真相源：5 家族 + 22 基线 + 9 条 L1 + 家族决策树
│   ├── layout.md                    # L1 基础栅格 + 跨家族规约
│   ├── component.md                 # L2 内容家族结构
│   ├── variant-evaluation.md        # 变体评估协议（100 分制 + 豁免流程）
│   ├── site-map.md                  # business_line + card_type 双标签体系
│   ├── competitors.md               # 竞品对比（小红书/抖音/INS/大众点评/美团/淘宝逛逛）
│   ├── jd-15-spec.md                # 15.0 内部规范（空壳，待注入）
│   └── changelog.md                 # 完整版本史
└── prompts/
    ├── page-level-review.md         # ★ agent 主入口：页面级审查执行契约（Step 0-6）
    ├── review-screenshots.md        # 单卡快检模板
    └── evaluate-variant.md          # 变体评估模板
```

---

## 三种工作流

| 场景 | 入口文件 | 典型触发语 |
|---|---|---|
| **页面级审查（主流）** | `prompts/page-level-review.md` | "帮我审查这个页面的双列卡" |
| 单卡快检 | `prompts/review-screenshots.md` | "这张截图有什么问题" |
| 变体评估 | `prompts/evaluate-variant.md` | "这个新变体做同物种判定" |

**如果 agent 只能读一个文件**：读 `prompts/page-level-review.md`（自包含执行契约）。

---

## 使用边界

### 能做
- 家族识别（视觉识别家族 + 子类型 + 媒介属性）
- L1 跨家族规约差异观察
- L2 家族专属规则观察（内容/点评/圈子/新品家族）
- 变体模式发现 + 跨场景同物种识别
- 流级信息类型一致性观察
- 规范反哺建议

### 不做
- 业务判断（该不该加引导、流程几步）
- 动态行为（加载态/错误态/空态/动效）
- 真实用户数据（CTR/停留时长/转化率）
- 用户上传的内容层评价（封面构图、标题文案好坏）
- 官方权威裁决（Skill 是辅助，最终裁决在评审委员会）

---

## 版本

当前版本：**v0.5.2**
状态：WIP
最近更新：2026-04-14

完整版本史见 [`references/changelog.md`](./references/changelog.md)。

---

## Owner

Shaka（京东 AI 体验设计 + 综合业务设计组）
