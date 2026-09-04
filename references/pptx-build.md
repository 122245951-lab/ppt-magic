# 用 python-pptx 生成可编辑 PPTX（工程参考）

> 本技能最终交付物之一是**真正可编辑的 .pptx**。本文给出生成脚本的骨架、主题色映射与常见坑。
> 反例（禁止）：HTML 网页幻灯片、长截图、把整页导出成图片贴进 PPT——这些都不可编辑，不算交付。

## 一、环境

- 用托管 Python 的隔离 venv（不要全局 pip）：
  - 解释器：`C:/Users/About/.workbuddy/binaries/python/envs/default/Scripts/python.exe`
  - 依赖：`python-pptx`（`.../Scripts/pip.exe install python-pptx`，已装则跳过）
- 脚本与产物放当前工作目录；产物命名 `<主题>-<场景>.pptx`。

## 二、16:9 画布与辅助函数骨架

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR
from pptx.enum.shapes import MSO_SHAPE

prs = Presentation()
prs.slide_width  = Inches(13.333)   # 16:9
prs.slide_height = Inches(7.5)
BLANK = prs.slide_layouts[6]        # 空白版式，所有元素自己画
SW, SH = prs.slide_width, prs.slide_height

def slide():                          # 新建一页 + 满版主题底色背景
    s = prs.slides.add_slide(BLANK)
    r = s.shapes.add_shape(MSO_SHAPE.RECTANGLE, 0, 0, SW, SH)
    r.fill.solid(); r.fill.fore_color.rgb = BG; r.line.fill.background()
    r.shadow.inherit = False
    return s

def box(s, x, y, w, h, fill=None, line=None, line_w=1.0, round_=False):
    shp = s.shapes.add_shape(
        MSO_SHAPE.ROUNDED_RECTANGLE if round_ else MSO_SHAPE.RECTANGLE,
        Inches(x), Inches(y), Inches(w), Inches(h))
    if fill is None: shp.fill.background()
    else: shp.fill.solid(); shp.fill.fore_color.rgb = fill
    if line is None: shp.line.fill.background()
    else: shp.line.color.rgb = line; shp.line.width = Pt(line_w)
    shp.shadow.inherit = False
    return shp

def R(t, sz, col, bold=False):       # 一个文字 run
    return (t, sz, col, bold)

def text(s, x, y, w, h, paragraphs, align=PP_ALIGN.LEFT,
         anchor=MSO_ANCHOR.TOP, space=1.1):
    # paragraphs: [[run, run, ...], ...]  外层每段一行
    tb = s.shapes.add_textbox(Inches(x), Inches(y), Inches(w), Inches(h))
    tf = tb.text_frame; tf.word_wrap = True; tf.vertical_anchor = anchor
    for i, para in enumerate(paragraphs):
        p = tf.paragraphs[0] if i == 0 else tf.add_paragraph()
        p.alignment = align; p.line_spacing = space
        for (t, sz, col, bold) in para:
            r = p.add_run(); r.text = t
            r.font.size = Pt(sz); r.font.color.rgb = col
            r.font.bold = bold; r.font.name = "微软雅黑"
    return tb

def notes(s, txt):                    # 演讲备注 → 写进每页 notes
    s.notes_slide.notes_text_frame.text = txt

def kicker(s, txt):                   # 页眉小标签
    box(s, 0.55, 0.5, 0.28, 0.045, fill=CYAN)
    text(s, 0.95, 0.34, 8, 0.4, [[R(txt, 13, CYAN, True)]])

def footer(s, left, no):
    text(s, 0.55, 7.02, 9, 0.35, [[R(left, 10.5, DIM, False)]])
    text(s, 12.0, 7.02, 0.9, 0.35, [[R(no, 11, CYAN, True)]], align=PP_ALIGN.RIGHT)

prs.save("输出.pptx")
```

要点：所有色块用 `add_shape` 矩形/圆角矩形，所有文字用独立 textbox，**最终每个字、每个色块在 PowerPoint/WPS/飞书幻灯片里都可点选编辑**。

## 三、主题色常量（与 style-presets.md 6 主题对应）

科技深色 B（默认深色）：
```
BG=0A1020  PANEL=121D3D  CYAN=22D3EE  BLUE=3B82F6  INDIGO=6366F1
TEXT=E8EEFC  MUTED=9FB0D0  DIM=6B7BA0  WARN=FBBF24  OK=34D399
```
其余 5 套主题取色见 `style-presets.md` 的色号；深色底用浅色文字（TEXT/MUTED），浅色底用深色文字。一页配色 ≤ 3 色。

## 四、页型 → 画法映射

- cover：kicker + 大标题（两行 run，关键词用 CYAN 高亮）+ 副题 + 元信息点。
- bullets/pain：竖排圆角卡片 `box(...)`，左缘 3px 强调色条 + emoji + 粗体小标题 + 说明。
- flow：横向 4 个 `box` 步骤卡，卡间用 textbox 放 "→"，步骤序号用 OVAL 圆。
- stat：3 列大数字卡，数字用 46–54pt CYAN，下面标签 + 小字来源。
- table：用 `add_table` 或用 box 拼行；标签（提效/辅助）用小圆角块 + OK/WARN 色。
- roadmap：左侧阶段色块（期次+周期）+ 右侧说明卡，竖排三行。
- closing：3 个申请项 box + 一句交付承诺。

## 五、保真与备注

- 数字/日期/专名按 `fidelity-and-editing.md` 建保真清单，脚本里这些字符串直接照抄来源，不做改写。
- 每页演讲词必须 `notes(s, "...")` 写入备注栏，与该页内容一致；数据来源写进备注。
- 成稿后自查：页数 = 大纲页数；每页关键数字与保真清单逐一对上。

## 六、上传飞书

```bash
# 注意：--file 用 cwd 相对路径；--type slides 导入为飞书幻灯片
lark-cli drive +import --file "./输出.pptx" --type slides --name "标题" --as user
# 返回 data.url 即分享链接；ready:false 时：
lark-cli drive +task_result --scenario import --ticket <TICKET>
```

## 七、常见坑

- **整页贴图**：不要把 HTML/图片塞进 PPT 当背景——不可编辑。一切用原生 shape/textbox。
- **文字溢出**：python-pptx 不自动分页，字号/框高要留余量；放不下就减字或拆页。
- **中文字体**：统一 `微软雅黑`（Windows/WPS/飞书均有），避免生僻字体回退乱版。
- **shadow.inherit=False**：新建 shape 默认带阴影和主题边框，显式关掉才干净。
- **绝对路径上传报错**：lark-cli `--file` 只接受相对路径，先 `cd` 到产物目录。
- **导入并发冲突**（错误码 232140101/232140100/233523001）：同一位置串行导入，失败等几秒重试，最多 3 次。
