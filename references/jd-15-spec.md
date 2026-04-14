# 京东主站 15.0 规范对接（空壳，待注入）

> **本文件等待 Shaka 注入内部规范**。目前只是一个占位结构，定义注入后的组织方式和冲突处理原则。
>
> 一旦 15.0 内部规范注入到本文件，**所有其他 references/ 文件中的数值以本文件为准**。

---

## 一、状态

- **当前状态**：空壳
- **预期注入内容**：京东主站 App 15.0 的权威设计 token 和双列卡相关规范
- **注入方式**：Shaka 把内部文档的关键数值粘贴到本文件对应章节
- **注入后动作**：Skill 自动执行冲突校验（见第六节）

---

## 二、待注入：设计 Token（Design Tokens）

### 2.1 颜色 Token

```
【待注入】
示例格式：
  primary.brand           = #fa2c19
  text.primary            = #1a1a1a
  text.secondary          = #666666
  text.tertiary           = #999999
  background.card         = #ffffff
  background.page         = #f5f5f5
  ...
```

### 2.2 字号 Token

```
【待注入】
示例格式：
  font.title.default      = 13px
  font.subtitle.default   = 12px
  font.meta.default       = 11px
  ...
```

### 2.3 间距 Token

```
【待注入】
示例格式：
  spacing.card.outer      = 8px
  spacing.card.inner      = 8px
  spacing.card.vertical   = 8px
  ...
```

### 2.4 圆角 Token

```
【待注入】
示例格式：
  radius.card             = 12px
  radius.image            = 8px
  radius.tag              = 12px
  radius.avatar           = 50%
  ...
```

---

## 三、待注入：双列卡专用规范

### 3.1 卡片宽度（v4.0 明确为 176dp，等 15.0 确认）

```
【待注入确认】
  cardWidth               = 176dp
  cardWidth.tolerance     = ±2dp
```

### 3.2 左右列高度差硬约束

```
【待注入确认】
  heightDiff.max          = 40px
```

### 3.3 图片比例约束

```
【待注入确认】
  image.ratio.default     = 3:4
  image.ratio.video       = 9:16
  image.ratio.allowed     = {3:4, 4:3, 1:1}
```

---

## 四、待注入：组件库映射

如果 15.0 有独立的 Figma 组件库/Storybook，以下字段需要填写：

- 组件库位置（Figma 链接 / 内部平台 URL）：— 
- 双列卡组件名称：—
- 相关 variants：—
- 负责人（组件库 owner）：—

---

## 五、待注入：15.0 改版相关文档

如果 15.0 有改版/迭代文档，可以附在此处：

- 改版背景与目标：—
- 决策记录（ADR）：—
- A/B 实验结果：—
- 上线时间：—

---

## 六、冲突处理原则

一旦本文件注入具体数值，本 Skill 的冲突处理规则：

### 规则 1：数值冲突 → 15.0 优先
如果 `jd-15-spec.md`（本文件）和其他 references 文件有数值冲突：
- **以本文件为准**
- 同时 **Edit 其他文件** 让它们对齐
- 在 `changelog.md` 记录冲突详情（谁改了谁、为什么）

### 规则 2：原则冲突 → SKILL.md 优先
如果本文件的数值对齐了，但和 `SKILL.md` 的核心判定原则有逻辑冲突：
- 以 `SKILL.md` 为准（原则层面不让步）
- 把冲突点上升给规范项目组，等官方明确后再调整

### 规则 3：注入之前的行为
注入前，本 Skill 按 v4.0 专项文档 + Shaka 口头补充（176dp 等）工作，已在其他 references 文件中明确标注"v4.0 权威值"。

### 规则 4：注入动作本身要留痕
每次注入动作（哪怕只是更新一个值）都要在 `changelog.md` 里记录：
- 时间、变更人、变更内容、是否触发其他文件调整、为什么

---

## 七、注入的操作建议（给 Shaka）

1. **整包注入**：把内部规范文档中与双列卡相关的 token / 规范 / 组件说明**整段复制**到对应章节
2. **先注入，后梳理**：原样粘贴，不要在注入过程中"顺手整理"——保持单一真相源
3. **冲突不要瞒**：发现 15.0 内部值和 v4.0 专项数值不一致，**不要静默对齐**，列出来讨论
4. **注入频率**：建议每个规范版本（大版本+小版本）都走一次注入，通过 `changelog.md` 版本号链条维护
