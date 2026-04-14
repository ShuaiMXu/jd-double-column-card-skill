# 页面级审查执行契约（Page-Level Review Contract）

> **面向 AI agent 执行的操作手册**。本文档的所有内容必须可直接映射到执行步骤，不做概念铺陈。
>
> **上游文件**（必读）：
> - `../SKILL.md`：定位与原则
> - `../references/business-baselines.md`：五大家族体系 + 22 个基线 + 11 条跨家族规约（**家族识别的真相源**）
> - `../references/layout.md`：L1 大规范
> - `../references/component.md`：L2 内容卡结构
>
> **版本**：v0.1（2026-04-14）

---

## 一、执行契约（Contract）

### 输入协议（Input Schema）

```yaml
page_review_request:
  page_screenshot:            # 必填
    file: <path>
  business_line:              # 必填，人工显式声明，Skill 不推断
    type: enum
    values: [推荐, 点评, 圈子, 新品, 个人中心, 直播]
  page_name:                  # 必填，自由文本
    example: "点评首页推荐流"
  version:                    # 可选
    type: enum
    values: [After, Before, 特定版本号]
  focus:                      # 可选
    type: string
    example: "重点看封面叠字合规性"
```

**输入缺失处理**：`business_line` 或 `page_screenshot` 缺失 → **拒绝执行**，返回错误提示要求补全。**不推断**。

### 输出协议（Output Schema）

```yaml
page_review_report:
  meta:
    review_date: <ISO 日期>
    skill_version: 0.5.0
    page_name: <string>
    business_line: <enum>
  section_1_overview:         # 页面摘要
    total_cards_detected: <int>
    family_distribution: <map>
    unknown_family_cards: <int>
  section_2_cards_by_family:  # 按家族分组的卡片清单
    - family: <enum>
      count: <int>
      sub_types: <list>
      cards: <list of card_analysis>
  section_3_observations:     # 差异观察（最多 8 条）
    - diff_description: <string>
      experience_cost: <string>
      possible_business_reason: <string>
      suggestion: <string>
  section_4_stream_level:     # 流级观察
    info_type_consistency: <string>
    family_distribution_reasonableness: <string>
  section_5_next_actions:     # 下一步建议（≤5 条）
    - action: <string>
      owner: <string>
      priority: [P0, P1, P2]
  section_6_limitations:      # 观察局限（必填）
    - <string>
```

---

## 二、执行流程（Step 0 → Step 6）

### Step 0 — 输入校验

```
IF business_line is null OR business_line not in [推荐,点评,圈子,新品,个人中心,直播]:
    RETURN error("business_line 必填且必须是合法枚举")
IF page_screenshot is null:
    RETURN error("page_screenshot 必填")
ELSE:
    PROCEED to Step 1
```

### Step 1 — 卡片切分（Card Segmentation）

在页面截图里识别出所有"双列卡"。

**识别规则**：
- 双列卡的视觉特征：**宽度约为屏宽的一半**（375px 屏下 ≈ 176dp/175px），**纵向堆叠排布**，**有清晰矩形边界**（圆角 ≈ 12px）
- **排除以下元素**：
  - 顶部导航栏、Tab 栏
  - 底部固定浮动按钮（发布 / 购物车等）
  - 单列大卡 / 横幅 Banner
  - 单行运营位（红包卡 / 活动卡等非典型双列卡结构）

**输出**：`total_cards_detected` 和每张卡的**截图位置坐标**（用于后续引用）。

**边界情况**：
- 如果页面完全看不到双列卡（比如这是一张详情页截图）→ **标记 "这不是双列卡页面"** 并终止执行
- 如果页面切分的卡数 < 2 → 依然执行但在报告中提示"样本量不足"

### Step 2 — 逐卡 3 步识别

对**每一张**在 Step 1 切出的卡，按以下 3 步识别：

#### Step 2a — 看底部结构识别家族（v0.5.2 修订版）

**这是核心指南针**。

**关键原则（v0.5.2）**：家族识别只依赖**结构不变量**（固定元素的有/无/位置），**不依赖动态数据**（种草数/观看量/点赞数等，可能为 0 → 不显示）。

按"结构簇"分层识别：

```
DECISION_TREE (修订版):

  CASE A: 底部行 contains [圆形头像 + 昵称 + 互动数据位]:
      # 内容家族和点评家族底部基础结构相同
      # 不能用"有无种草数"判定（种草数是动态数据，0 人种草时就不显示）
      # 靠 business_line 辅助区分
      IF business_line IN {点评, 个人中心}:
          family = 点评
      ELSE IF business_line == 推荐:
          family = 内容
      ELSE:
          family = 内容/点评
          confidence = low

      # 附加元素作为"激活信号"记录，不影响家族判定
      annotations:
        has_seed_count: <bool>       # 标题下方是否有"X 人种草"绿✓
        has_identity_badge: <bool>   # 昵称旁是否有身份胶囊（如"求真官"）
        interaction_icon: <enum>     # ❤️ | 👁 | 💬 | ⭐

  CASE B: 底部行 contains [店铺 logo + 店铺名 + 价格/销量/促销]:
      family = 商品

  CASE C: 底部行 contains [圈子图标 + 圈子名]  AND  右下为空:
      family = 圈子

  CASE D: 底部 contains [
      CTA 按钮(如"XX 限定") OR
      事件胶囊(如"重磅发布") OR
      白底商品挂载浮条(含商品名+描述) OR
      话题标签(如"#XXX") + 话题热度
  ]  AND  没有作者/店铺/圈子信息:
      family = 新品

  ELSE:
      family = 未知
      标记 card 为 "unknown_family_card"
      纳入 section_1_overview.unknown_family_cards 计数
```

**特殊混血场景**：
- **种草卡**（左上话题标签 + 封面下商品挂载浮条 + 底部 UGC）：family = 点评（或"内容·种草子分支"，取决于业务方定义；baselines 暂归点评家族附近）
- **明星同款卡**（封面是明星 + 底部商品挂载浮条 + 无作者显式）：family = 新品（隐形作者特例）

**动态数据 vs 结构不变量 速查表**：

| 元素 | 性质 | 是否参与家族识别 |
|---|---|---|
| 头像 + 昵称（占位） | 结构 | ✓ 是 |
| 互动数据位（有图标位置）| 结构 | ✓ 是 |
| 店铺 logo + 店铺名 | 结构 | ✓ 是 |
| 圈子图标 + 圈子名 | 结构 | ✓ 是 |
| 互动数据的具体数字 | 动态 | ❌ 否 |
| **种草数的有无**（0 人时不显示）| 动态 | ❌ 否 |
| 观看量的有无 | 动态 | ❌ 否 |
| 身份标签（求真官等） | 结构 | ✓ 是（家族激活信号）|

#### Step 2b — 看封面+标题识别子类型

按 `business-baselines.md` 第三节的子类型矩阵，逐项匹配：

```
子类型识别逻辑（伪代码）:

SWITCH family:
    CASE 内容:
        IF 左上角 = 红色直播图标:
            子类型候选 = [内容-直播卡基础 | 红包雨变体 | 超头/达人变体 | 预约变体 | 沉浸式变体]
            进一步看 封面/右上角/标题下方 区分
        ELSE IF 左上角 = 视频角标:
            子类型 = 内容-视频卡
        ELSE:
            子类型 = 内容-图文卡（互动子样式看右下图标决定: 点赞/观看/评论/收藏）

    CASE 商品:
        IF 左上角 = 红色直播图标:
            子类型候选 = [商品-直播卡基础 | 弹幕外露 | 沉浸式商卡 | 沉浸式促销]
        ELSE IF 左上角 = 视频角标:
            子类型 = 商品-视频卡
        ELSE:
            子类型 = 商品-图文卡（当前 baselines 缺基础款基线）

    CASE 圈子:
        IF 左上角 = 视频角标:
            子类型 = 圈子-视频卡
        ELSE:
            子类型 = 圈子-图文卡

    CASE 点评:
        IF 左上角 = 视频角标:
            子类型 = 点评-视频卡
        ELSE IF 昵称右侧 = 红色身份胶囊:
            子类型 = 点评-身份标签变体
        ELSE:
            子类型 = 点评-图文卡

    CASE 新品:
        IF 左上角 = "新发布 + MM/DD" 红胶囊:
            子类型 = 新品-入口卡
        ELSE IF 左上角 = 红色直播图标:
            子类型 = 新品-直播卡入口
        ELSE IF 左上角 = 视频角标 + 时长:
            子类型 = 新品-测评视频卡
        ELSE IF 封面 = 多图拼图集锦 AND 底部 = 话题标签+热度:
            子类型 = 新品-趋势专属图文卡
        ELSE IF 封面 = 明星人像 AND 底部 = 商品挂载+种草数:
            子类型 = 新品-明星同款卡
        ELSE:
            子类型 = 新品-未知子类型（纳入观察）
```

#### Step 2c — 看左上角确认媒介属性

```
媒介属性识别:

  IF 左上角 contains 红色直播图标:
      media = 直播
      IF 左上角 contains 观看数(如 "X 万观看"):
          additional_attr = 带热度
  ELSE IF 左上角 contains 白色播放 ▶ 角标:
      media = 视频
      IF 左上角 contains 时长(如 "00:10"):
          additional_attr = 带时长
  ELSE IF 左上角 contains 图文 icon:
      media = 图文
      business_line MUST be 个人中心  # 图文 icon 仅个人页显示
  ELSE:
      media = 图文（默认态）
```

### Step 3 — 对照 business_line 检查"家族分布合理性"

对照 `business_line` 与实际识别出的 `family_distribution`，检查是否出现**非预期家族**。

**预期家族对照表**（基于 business-baselines.md 常识）：

| business_line | 主体家族 | 允许出现的其他家族 | 非预期家族（值得讨论） |
|---|---|---|---|
| 推荐 | 内容 | 少量商品/新品/点评 | 圈子 |
| 点评 | 点评 | 内容 | 商品 / 圈子 / 新品 |
| 圈子 | 圈子 | — | 其他都算非预期 |
| 新品 | 新品 | 少量内容/商品 | 圈子 / 点评 |
| 个人中心 | 点评（种草变体）/ 内容 | 圈子 / 新品 | 商品 |
| 直播 | 内容（直播）/ 商品（直播） | 运营位（不纳入审查）| 圈子 / 新品入口卡（非直播形式） |

**输出**：如果发现"非预期家族"，纳入 `section_4_stream_level.family_distribution_reasonableness` 作为讨论项，**不作为违规判决**。

### Step 4 — 按家族分组走 L1 + L2 规则

对每张卡，按照 baselines 的规约进行**差异观察**（不是违规判定）。

#### ★ v0.5.2 执行约束：结构优先、内容无涉

**Skill 只观察设计师规范可控的结构性元素，不观察用户/运营上传的内容层**。

**观察范围**（Structure，要看）：
- 框体尺寸、圆角、间距、栅格、布局
- 字号、字重、颜色 token
- 组件有无（头像、标签、互动数据位等）
- 元素位置（左上/右上/标题下/底部）
- 层级结构

**不观察**（Content，不要看）：
- 封面图具体内容、构图、比例偏向（偏横/偏方**都不判定为差异**）
- 标题具体文案质量（只看字数合规性，不评价文案写得好不好）
- 头像具体图像
- 运营配图色彩倾向

**例外**：图片比例明显超出规范值域（如 1:2 极端比例，属规范违规）要报。值域内不同选择（3:4 vs 4:3）不属于差异。

**典型错误示例**：
- ❌ "Card X 酒店图片偏横 → 瀑布流节奏不齐" （这是内容层，用户上传的图就是横图，设计师无法控制）
- ✅ "Card X 图片比例 1:2 → 超出规范值域 {3:4, 4:3, 1:1}" （这是值域违规，要报）

#### L1 观察（所有卡）

对照 `business-baselines.md` 第二节的 **L1-1 到 L1-9** 规约，识别差异：

```
FOR EACH card in all_cards:
    FOR EACH rule in [L1-1, L1-2, L1-3, L1-4, L1-5, L1-6, L1-7, L1-8, L1-9]:
        IF card violates rule:
            记录为一个 observation:
                diff_description = 具体差异
                experience_cost = 从"体验视角工具箱"里挑 1 个最相关的维度展开
                possible_business_reason = 业务方可能的考量（要猜得有诚意）
                suggestion = 讨论方向
```

**L1 规约速查**：
- L1-1 左上角 = 卡片属性表达位
- L1-2 右上角 = 辅助信号位（可为空）
- L1-3 标题下方 = 关于"内容"的元信息位
- L1-4 商品挂载浮条位置 = 反映叙事方向
- L1-5 **底部不变量 = 家族 DNA 锚定**
- L1-6 内容卡右下互动信号位可切换
- L1-7 封面浮层 = 决策信息前置位（描述性）
- L1-8 梯度叠加封面 = 一种封面结构类型（描述性）
- L1-9 昵称旁 = 关于"人"的元信息位

#### L2 观察（按家族分支）

按识别出的 `family`，走对应的 L2 规则：
- 内容家族 → `references/component.md` 第二/三/四节（图文/视频/直播结构）
- 商品家族 → 暂只走 L1（L2 待补）
- 圈子家族 → baselines 第 3.3 节
- 点评家族 → baselines 第 3.4 节
- 新品家族 → baselines 第 3.5 节（按子类型）

### Step 5 — 流级检查

```
stream_level_checks:

  信息类型一致性:
      扫描所有 same_business_line 的卡, 判断是否存在叙事混杂
      例如: 点评流里同时出现"店铺种草"叙事 + "UGC 测评"叙事
      IF 混杂:
          记录到 section_4.info_type_consistency
          体验评价: "用户扫视时需反复切换'谁在说'上下文"

  视觉风格统一性:
      同业务线的卡, 视觉调性是否协调
      IF 风格断裂(如部分卡纯净+部分卡红色氛围):
          记录到 section_4

  家族分布合理性:
      见 Step 3 结果
```

### Step 6 — 输出报告

使用下一节的**报告模板**，按 section_1 到 section_6 填写。

---

## 三、报告模板（精简版，一屏能看完）

> 按 v0.3.1 确立的"简单优先"原则。整份报告不超过 1000 字。

```markdown
# 双列卡页面观察 - {{YYYY-MM-DD}}

**页面**：{{page_name}} | **业务线**：{{business_line}} | **skill 版本**：v{{version}}

## 一、页面摘要
- 识别卡片总数：{{N}}
- 家族分布：{{family_distribution as table}}
- 未识别家族卡：{{unknown_count}}（如有，列出位置）

## 二、卡片家族分布

| 家族 | 数量 | 子类型明细 |
|---|---|---|
| ... | ... | ... |

## 三、差异观察（≤ 8 条，只写有差异的）

| # | 差异 | 体验代价 | 可能的业务理由 | 建议 |
|---|---|---|---|---|
| 1 | ... | ... | ... | ... |

## 四、流级观察
- 信息类型一致性：{{评价}}
- 家族分布合理性：{{评价，含非预期家族如有}}

## 五、下一步建议（≤ 5 条）
- [ ] {{行动项，含 owner 和 priority}}

## 六、观察局限（诚实声明）
- 静态截图看不到动效 / 加载态 / 错误态
- VLM 视觉估算存在误差
- 未覆盖深色模式 / 长辈版
- {{其他本次具体局限}}
```

---

## 四、边界与兜底

### 4.1 家族识别失败

如果 Step 2a 决策树走到 `family = 未知`：
1. 不强行归类
2. 标记该卡为 "unknown_family_card"，记录截图位置
3. 在报告 section_1 中 count
4. 在 section_5 下一步建议中追加：**"识别到 N 张未知家族卡，建议业务方确认该卡类型归属或反哺 baselines"**

### 4.2 业务线 vs 家族冲突

调用方声明 `business_line = 推荐`，但 Step 2a 识别出大量 `family = 圈子` 的卡 → **不要否决调用方声明**，而是：
1. 保持 business_line = 推荐（按声明走）
2. 在 section_4 记录"推荐页出现大量圈子卡"作为流级观察
3. 让业务方自己判断是"误分发"还是"有意混排"

### 4.3 非本 Skill 范围的卡

页面里出现：
- 单列大卡 / 横幅 Banner：Step 1 已排除
- 商品卡 / 运营位：Step 4 只走 L1，不走 L2（L2 待补规范）
- **完全不是卡的元素**（如模态浮层、搜索框）：Step 1 已排除

### 4.4 识别置信度低

某张卡的识别信号模糊（比如图太小、遮挡严重）：
1. 在 "card_analysis" 中标注 `confidence: low`
2. 报告中明确标"识别置信度低"
3. 建议用户重新提供更清晰截图

---

## 五、与其他 Skill 文件的关系（给 agent 的索引）

| 需要判断 | 查哪个文件 |
|---|---|
| 家族识别规则 | `../references/business-baselines.md` 第四节 |
| 22 个子类型基线 | `../references/business-baselines.md` 第三节 |
| 11 条跨家族规约（L1）| `../references/business-baselines.md` 第二节 |
| 内容卡 L2 专项 | `../references/component.md` |
| 基础栅格/间距/圆角 L1 | `../references/layout.md` |
| 有效豁免清单 | `../references/changelog.md` 末尾 |
| 变体评估协议 | `../references/variant-evaluation.md` |
| 观察原则（不判决）| `../SKILL.md` 第三节 |

---

## 六、执行示例

### 输入
```yaml
page_screenshot: /path/to/dianping-home.png
business_line: 点评
page_name: 点评首页推荐流
```

### 执行轨迹
```
Step 0 ✓ 输入完整
Step 1 ✓ 识别 8 张双列卡
Step 2 (对每张卡):
  card_1 底部=头像+昵称+互动+标题下"12.3万人种草" → family=点评, 子类型=点评-图文卡, media=图文
  card_2 同上 → family=点评
  card_3 昵称旁"求真官"胶囊 → family=点评, 子类型=点评-身份标签变体
  ...
Step 3 ✓ 家族分布 = {点评: 8}，符合 business_line=点评 预期
Step 4 L1/L2 差异观察:
  - 发现 card_5 图片比例为 4:3（其他都是 3:4）→ L1 差异
  - 发现 card_7 标题溢出 3 行 → L2 差异
Step 5 流级观察:
  - 同业务线叙事混合（店铺种草 + UGC 测评）→ 信息类型一致性问题
Step 6 输出报告
```

### 输出（片段）
```markdown
# 双列卡页面观察 - 2026-04-14

**页面**：点评首页推荐流 | **业务线**：点评 | **skill 版本**：v0.5.0

## 一、页面摘要
- 识别卡片总数：8
- 家族分布：点评(8)
- 未识别家族卡：0

## 三、差异观察
| # | 差异 | 体验代价 | 可能的业务理由 | 建议 |
|---|---|---|---|---|
| 1 | card_5 图片比例 4:3 (其他 3:4) | 流内视觉不齐, 扫视疲劳 | 内容本身是横图 | 讨论统一比例或允许横图 |
| 2 | card_7 标题溢出 3 行 | 标题占用空间过大, 封面被挤压 | 标题过长未省略 | 文案压缩至 24 字内 |
...
```

---

## 七、版本

| 版本 | 日期 | 更新内容 |
|---|---|---|
| v0.1 | 2026-04-14 | 初版，基于 business-baselines.md v0.3 创建页面级审查执行契约 |
