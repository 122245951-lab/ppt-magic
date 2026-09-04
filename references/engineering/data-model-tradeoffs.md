# 数据模型取舍：结构化内容寻址 vs 自由画布

> 演示文档类产品第一个架构决策。本文给出一份经过实战验证的取舍论证，以及"结构化内容模型 + 锁定版式渲染"的具体规范。评审产品方案时可直接引用。

## 一、两种范式

| 维度 | A. 结构化内容模型（本蓝图采用） | B. 自由画布（像素坐标） |
|---|---|---|
| 文档表示 | `slides[i].content.<field>`（title/items/stats/chart…） | `elements[]`，每元素带 x/y/w/h/z |
| AI 编辑 | 对字段出 ops，天然局部、可 diff、可撤销 | 要定位"改哪个元素"，语义与坐标两层 |
| 排版质量 | 版式库渲染，下限有保证 | 依赖模型每次给对坐标，易破版 |
| 局部编辑 | 元素=内容字段，指向即达 | 需要框选坐标↔语义元素映射层 |
| 撤销/保真 | 字段级 forward/inverse，精确 | 整树快照为主，细粒度难 |
| 代价 | 表达自由受限（不适合超复杂自绘布局） | 表达自由（代价是质量失控与实现复杂度） |

**结论**：面向"汇报/商业演示"的 Copilot，选 A。用户要的是"专业、不出格、改得动"，不是"像素级自绘"。"会排版"恰恰来自限制表达空间。

## 二、内容寻址规范

```
slides[i].content.title            # 标题
slides[i].content.items[k].text    # 第 k 条要点文本
slides[i].content.items[k].detail  # 要点补充
slides[i].content.stats[k].value   # 指标数字（保真重点）
slides[i].content.columns[j].items[m]
slides[i].content.chart            # 图表对象
slides[i].variant                  # 锁定版式 id（只能取版式库枚举）
slides[i].design.typography.headline  # 标题字号档
```

- **每元素可携带 `scale?: number`**：元素级字号倍率（放大这条、不碰全局），是"局部编辑"与"全局字号档"之间的中间形态，强烈建议保留。
- **渲染层单向映射**：content/design → 版式渲染；渲染不做逆向推断，保证"所见即存"。
- **普通文本与富文本**：文本存纯字符串 + 可选占位语义，不做段落内 rich model（保真指纹与差分都建立在字符串层，简单可靠）。

## 三、路径白名单（写保护规则）

所有修改必须经过 patch 引擎，其路径白名单如下，违反即整批拒绝：

1. 只允许写 `slides[i].content.*`、`slides[i].design.*`、`slides[i].variant`、`title/themeId/fidelityLevel` 等白名单根。
2. **禁写结构字段**：`id`、`type`（页型根），防止 AI 破坏数据模型约束。
3. 示例：不能直接写 `slides[0].content.chart.type`（`type` 被禁），正确姿势是整体替换 `chart` 对象 `{ ...chart, type: 'line' }`。
4. 每个 op 可生成 inverse（原值/原位），供统一撤销使用。

## 四、"背景渐变"这类页面级诉求怎么归类

自由画布语境下的"圈选改背景为浅蓝渐变"在结构化模型里本质是**页面级/主题级视觉属性**，不属于元素局部编辑。规约建议：

- 局部编辑通道只受理"内容字段 + 元素级 scale"。
- 页面级诉求（背景、主题、整页版式）路由到"当前页/全篇"作用域，走影响清单确认。
- 不要为了一个词兼容"像素级自由"，把架构拉回范式 B —— 明确能力边界本身就是产品表达。

## 五、迁移与扩展提示

- 旧库结构演进用幂等迁移（`PRAGMA table_info` 检测缺列 → `ALTER TABLE ADD COLUMN`），索引在补列之后再建，否则 `CREATE INDEX` 在旧表上直接报 no such column。
- 保留"每元素 scale""diff 标注"等字段时，给渲染层透传 `style` 的钩子即可，别把视觉写死在版式组件里。
