---
name: docx
description: "当用户想要创建、读取、编辑或处理 Word 文档（.docx 文件）时使用此 skill。触发场景包括：任何提到 'Word doc'、'word document'、'.docx'，或要求生成带有目录、标题、页码、信头等格式的专业文档。也适用于从 .docx 文件中提取或重组内容、在文档中插入或替换图片、对 Word 文件执行查找替换、处理修订或批注，或将内容转换为排版完善的 Word 文档。如果用户要求生成“报告”“备忘录”“信函”“模板”或类似的 Word / .docx 交付物，请使用此 skill。不要将其用于 PDF、电子表格、Google Docs，或与文档生成无关的一般编程任务。"
license: 专有。完整条款见 LICENSE.txt
---

# DOCX 的创建、编辑与分析

## 概述

.docx 文件本质上是一个包含 XML 文件的 ZIP 压缩包。

## 快速参考

| 任务 | 处理方式 |
|------|----------|
| 读取/分析内容 | 使用 `pandoc`，或解包以查看原始 XML |
| 创建新文档 | 使用 `docx-js`，见下文“创建新文档” |
| 编辑现有文档 | 解包 → 编辑 XML → 重新打包，见下文“编辑现有文档” |

### 将 .doc 转换为 .docx

旧版 `.doc` 文件在编辑前必须先转换：

```bash
python scripts/office/soffice.py --headless --convert-to docx document.doc
```

### 读取内容

```bash
# 提取包含修订痕迹的文本
pandoc --track-changes=all document.docx -o output.md

# 访问原始 XML
python scripts/office/unpack.py document.docx unpacked/
```

### 转换为图片

```bash
python scripts/office/soffice.py --headless --convert-to pdf document.docx
pdftoppm -jpeg -r 150 document.pdf page
```

### 接受修订

如果要生成一个“已接受所有修订”的干净文档（需要 LibreOffice）：

```bash
python scripts/accept_changes.py input.docx output.docx
```

---

## 创建新文档

使用 JavaScript 生成 .docx 文件，然后进行校验。安装方式：`npm install -g docx`

### 基本用法
```javascript
const { Document, Packer, Paragraph, TextRun, Table, TableRow, TableCell, ImageRun,
        Header, Footer, AlignmentType, PageOrientation, LevelFormat, ExternalHyperlink,
        InternalHyperlink, Bookmark, FootnoteReferenceRun, PositionalTab,
        PositionalTabAlignment, PositionalTabRelativeTo, PositionalTabLeader,
        TabStopType, TabStopPosition, Column, SectionType,
        TableOfContents, HeadingLevel, BorderStyle, WidthType, ShadingType,
        VerticalAlign, PageNumber, PageBreak } = require('docx');

const doc = new Document({ sections: [{ children: [/* 内容 */] }] });
Packer.toBuffer(doc).then(buffer => fs.writeFileSync("doc.docx", buffer));
```

### 校验
创建文件后要进行校验。如果校验失败，就解包、修复 XML，再重新打包。
```bash
python scripts/office/validate.py doc.docx
```

### 页面尺寸

```javascript
// 关键：docx-js 默认是 A4，不是 US Letter
// 为保证结果一致，始终要显式设置页面尺寸
sections: [{
  properties: {
    page: {
      size: {
        width: 12240,   // 8.5 英寸，单位为 DXA
        height: 15840   // 11 英寸，单位为 DXA
      },
      margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } // 1 英寸页边距
    }
  },
  children: [/* 内容 */]
}]
```

**常见纸张尺寸（DXA 单位，1440 DXA = 1 英寸）：**

| 纸张 | 宽度 | 高度 | 内容宽度（1 英寸页边距） |
|-------|-------|--------|---------------------------|
| US Letter | 12,240 | 15,840 | 9,360 |
| A4（默认） | 11,906 | 16,838 | 9,026 |

**横向页面：** docx-js 会在内部交换宽高，因此应传入纵向尺寸，由它自行处理交换：
```javascript
size: {
  width: 12240,   // 宽度传短边
  height: 15840,  // 高度传长边
  orientation: PageOrientation.LANDSCAPE  // docx-js 会在 XML 中交换它们
},
// 内容宽度 = 15840 - 左边距 - 右边距（使用长边）
```

### 样式（覆盖内置标题样式）

默认字体使用 Arial（通用支持最好）。标题保持黑色以保证可读性。

```javascript
const doc = new Document({
  styles: {
    default: { document: { run: { font: "Arial", size: 24 } } }, // 默认 12pt
    paragraphStyles: [
      // 重要：使用精确的 ID 才能覆盖内置样式
      { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 32, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 240, after: 240 }, outlineLevel: 0 } }, // TOC 需要 outlineLevel
      { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal", quickFormat: true,
        run: { size: 28, bold: true, font: "Arial" },
        paragraph: { spacing: { before: 180, after: 180 }, outlineLevel: 1 } },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ heading: HeadingLevel.HEADING_1, children: [new TextRun("标题")] }),
    ]
  }]
});
```

### 列表（绝不要手写 Unicode 项目符号）

```javascript
// ❌ 错误：绝不要手动插入项目符号字符
new Paragraph({ children: [new TextRun("• 项目")] })  // 错误
new Paragraph({ children: [new TextRun("\u2022 项目")] })  // 错误

// ✅ 正确：使用 numbering 配置和 LevelFormat.BULLET
const doc = new Document({
  numbering: {
    config: [
      { reference: "bullets",
        levels: [{ level: 0, format: LevelFormat.BULLET, text: "•", alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } }] },
      { reference: "numbers",
        levels: [{ level: 0, format: LevelFormat.DECIMAL, text: "%1.", alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } }] },
    ]
  },
  sections: [{
    children: [
      new Paragraph({ numbering: { reference: "bullets", level: 0 },
        children: [new TextRun("项目符号项")] }),
      new Paragraph({ numbering: { reference: "numbers", level: 0 },
        children: [new TextRun("编号项")] }),
    ]
  }]
});

// ⚠️ 每个 reference 都会创建独立编号序列
// 相同 reference = 继续编号（1,2,3 然后 4,5,6）
// 不同 reference = 重新开始（1,2,3 然后 1,2,3）
```

### 表格

**关键：表格需要双重宽度设置** - 既要在表格上设置 `columnWidths`，也要在每个单元格上设置 `width`。少任何一个，某些平台上的渲染都会出错。

```javascript
// 关键：始终设置表格宽度，保证渲染一致
// 关键：使用 ShadingType.CLEAR（不要用 SOLID），避免背景变黑
const border = { style: BorderStyle.SINGLE, size: 1, color: "CCCCCC" };
const borders = { top: border, bottom: border, left: border, right: border };

new Table({
  width: { size: 9360, type: WidthType.DXA }, // 始终使用 DXA（百分比在 Google Docs 中会失效）
  columnWidths: [4680, 4680], // 总和必须等于表格宽度（DXA: 1440 = 1 英寸）
  rows: [
    new TableRow({
      children: [
        new TableCell({
          borders,
          width: { size: 4680, type: WidthType.DXA }, // 每个单元格也必须设置
          shading: { fill: "D5E8F0", type: ShadingType.CLEAR }, // 用 CLEAR，不用 SOLID
          margins: { top: 80, bottom: 80, left: 120, right: 120 }, // 单元格内边距（内部，不会增加宽度）
          children: [new Paragraph({ children: [new TextRun("单元格")] })]
        })
      ]
    })
  ]
})
```

**表格宽度计算：**

始终使用 `WidthType.DXA`，`WidthType.PERCENTAGE` 在 Google Docs 中会出问题。

```javascript
// 表格宽度 = columnWidths 之和 = 内容宽度
// US Letter 且页边距为 1 英寸：12240 - 2880 = 9360 DXA
width: { size: 9360, type: WidthType.DXA },
columnWidths: [7000, 2360]  // 总和必须等于表格宽度
```

**宽度规则：**
- **始终使用 `WidthType.DXA`** - 不要使用 `WidthType.PERCENTAGE`（与 Google Docs 不兼容）
- 表格宽度必须等于 `columnWidths` 之和
- 单元格 `width` 必须与对应的 `columnWidth` 一致
- 单元格 `margins` 是内部留白，会减少内容区，不会增加单元格宽度
- 全宽表格应使用内容宽度（页面宽度减去左右页边距）

### 图片

```javascript
// 关键：type 参数是必填项
new Paragraph({
  children: [new ImageRun({
    type: "png", // 必填：png、jpg、jpeg、gif、bmp、svg
    data: fs.readFileSync("image.png"),
    transformation: { width: 200, height: 150 },
    altText: { title: "标题", description: "描述", name: "名称" } // 三项都必填
  })]
})
```

### 分页符

```javascript
// 关键：PageBreak 必须放在 Paragraph 内部
new Paragraph({ children: [new PageBreak()] })

// 或使用 pageBreakBefore
new Paragraph({ pageBreakBefore: true, children: [new TextRun("新的一页")] })
```

### 超链接

```javascript
// 外部链接
new Paragraph({
  children: [new ExternalHyperlink({
    children: [new TextRun({ text: "点击这里", style: "Hyperlink" })],
    link: "https://example.com",
  })]
})

// 内部链接（书签 + 引用）
// 1. 在目标位置创建书签
new Paragraph({ heading: HeadingLevel.HEADING_1, children: [
  new Bookmark({ id: "chapter1", children: [new TextRun("第 1 章")] }),
]})
// 2. 链接到该书签
new Paragraph({ children: [new InternalHyperlink({
  children: [new TextRun({ text: "查看第 1 章", style: "Hyperlink" })],
  anchor: "chapter1",
})]})
```

### 脚注

```javascript
const doc = new Document({
  footnotes: {
    1: { children: [new Paragraph("来源：2024 年年度报告")] },
    2: { children: [new Paragraph("方法说明见附录")] },
  },
  sections: [{
    children: [new Paragraph({
      children: [
        new TextRun("营收增长了 15%"),
        new FootnoteReferenceRun(1),
        new TextRun("，使用的是调整后口径"),
        new FootnoteReferenceRun(2),
      ],
    })]
  }]
});
```

### 制表位

```javascript
// 在同一行中将文本右对齐（例如：标题左侧，日期右侧）
new Paragraph({
  children: [
    new TextRun("公司名称"),
    new TextRun("\t2025 年 1 月"),
  ],
  tabStops: [{ type: TabStopType.RIGHT, position: TabStopPosition.MAX }],
})

// 点引导线（例如目录样式）
new Paragraph({
  children: [
    new TextRun("引言"),
    new TextRun({ children: [
      new PositionalTab({
        alignment: PositionalTabAlignment.RIGHT,
        relativeTo: PositionalTabRelativeTo.MARGIN,
        leader: PositionalTabLeader.DOT,
      }),
      "3",
    ]}),
  ],
})
```

### 多栏布局

```javascript
// 等宽分栏
sections: [{
  properties: {
    column: {
      count: 2,          // 栏数
      space: 720,        // 栏间距，单位 DXA（720 = 0.5 英寸）
      equalWidth: true,
      separate: true,    // 栏之间显示竖线
    },
  },
  children: [/* 内容会自然流入各栏 */]
}]

// 自定义栏宽（equalWidth 必须为 false）
sections: [{
  properties: {
    column: {
      equalWidth: false,
      children: [
        new Column({ width: 5400, space: 720 }),
        new Column({ width: 3240 }),
      ],
    },
  },
  children: [/* 内容 */]
}]
```

如果要强制换栏，可新建一个 section，并使用 `type: SectionType.NEXT_COLUMN`。

### 目录

```javascript
// 关键：标题必须只使用 HeadingLevel，不能额外套自定义样式
new TableOfContents("目录", { hyperlink: true, headingStyleRange: "1-3" })
```

### 页眉/页脚

```javascript
sections: [{
  properties: {
    page: { margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } } // 1440 = 1 英寸
  },
  headers: {
    default: new Header({ children: [new Paragraph({ children: [new TextRun("页眉")] })] })
  },
  footers: {
    default: new Footer({ children: [new Paragraph({
      children: [new TextRun("第 "), new TextRun({ children: [PageNumber.CURRENT] }), new TextRun(" 页")]
    })] })
  },
  children: [/* 内容 */]
}]
```

### docx-js 的关键规则

- **显式设置页面尺寸** - docx-js 默认使用 A4；对于美式文档应使用 US Letter（12240 x 15840 DXA）
- **横向页面要传纵向尺寸** - docx-js 会在内部交换宽高；传入短边作为 `width`、长边作为 `height`，并设置 `orientation: PageOrientation.LANDSCAPE`
- **不要使用 `\n`** - 应拆成多个 Paragraph 元素
- **不要手写 Unicode 项目符号** - 使用带 `LevelFormat.BULLET` 的 numbering 配置
- **PageBreak 必须位于 Paragraph 中** - 单独使用会生成无效 XML
- **ImageRun 必须包含 `type`** - 始终显式指定 png/jpg 等格式
- **表格 `width` 必须使用 DXA** - 不要使用 `WidthType.PERCENTAGE`（会在 Google Docs 中失效）
- **表格需要双重宽度** - `columnWidths` 数组和单元格 `width` 都必须设置且一致
- **表格宽度必须等于 `columnWidths` 之和** - 使用 DXA 时必须保证精确相加
- **始终添加单元格边距** - 使用 `margins: { top: 80, bottom: 80, left: 120, right: 120 }` 以获得合适留白
- **使用 `ShadingType.CLEAR`** - 表格底纹不要使用 SOLID
- **不要用表格充当分隔线/横线** - 单元格有最小高度，会渲染成空框（页眉页脚里也一样）；应改为在 Paragraph 上使用 `border: { bottom: { style: BorderStyle.SINGLE, size: 6, color: "2E75B6", space: 1 } }`。如果页脚需要两列布局，应使用制表位（见“制表位”），不要使用表格
- **TOC 只能依赖 HeadingLevel** - 标题段落上不要再叠加自定义样式
- **覆盖内置标题样式时必须用精确 ID** - 如 "Heading1"、"Heading2"
- **必须包含 `outlineLevel`** - 目录需要它（H1 为 0，H2 为 1，以此类推）

---

## 编辑现有文档

**严格按顺序执行以下 3 个步骤。**

### 第 1 步：解包
```bash
python scripts/office/unpack.py document.docx unpacked/
```
这会提取 XML、进行美化格式化、合并相邻 run，并把智能引号转换为 XML 实体（如 `&#x201C;`），以免编辑时丢失。如果想跳过 run 合并，可使用 `--merge-runs false`。

### 第 2 步：编辑 XML

编辑 `unpacked/word/` 下的文件。具体模式见下文“XML 参考”。

**修订和批注的作者默认使用 "Claude"**，除非用户明确要求使用其他名字。

**做字符串替换时，直接使用编辑工具，不要写 Python 脚本。** 脚本会引入不必要的复杂度，而编辑工具能明确展示替换内容。

**关键：新增内容要使用智能引号。** 当添加包含撇号或引号的文本时，应使用 XML 实体生成专业排版效果：
```xml
<!-- 使用这些实体来获得专业排版效果 -->
<w:t>这是一句带引号的话：&#x201C;你好&#x201D;</w:t>
```
| 实体 | 字符 |
|--------|-----------|
| `&#x2018;` | ‘（左单引号） |
| `&#x2019;` | ’（右单引号 / 撇号） |
| `&#x201C;` | “（左双引号） |
| `&#x201D;` | ”（右双引号） |

**添加批注：** 使用 `comment.py` 处理多个 XML 文件中的样板内容（文本必须是预先转义好的 XML）：
```bash
python scripts/comment.py unpacked/ 0 "批注文本，包含 &amp; 和 &#x2019;"
python scripts/comment.py unpacked/ 1 "回复文本" --parent 0  # 回复批注 0
python scripts/comment.py unpacked/ 0 "文本" --author "自定义作者"  # 自定义作者名称
```
然后再把标记插入到 document.xml 中（见“XML 参考”里的“批注”部分）。

### 第 3 步：重新打包
```bash
python scripts/office/pack.py unpacked/ output.docx --original document.docx
```
该步骤会执行校验、自动修复、压缩 XML，并生成 DOCX。若要跳过校验，可使用 `--validate false`。

**自动修复会处理：**
- `durableId` >= 0x7FFFFFFF（会重新生成合法 ID）
- `<w:t>` 在有空白字符时缺少 `xml:space="preserve"`

**自动修复不会处理：**
- XML 格式错误、非法元素嵌套、缺失关系、Schema 校验错误

### 常见陷阱

- **替换整个 `<w:r>` 元素**：在添加修订时，要把整个 `<w:r>...</w:r>` 替换成作为同级节点的 `<w:del>...<w:ins>...`。不要把修订标签塞进一个 run 的内部。
- **保留 `<w:rPr>` 格式信息**：将原 run 的 `<w:rPr>` 块复制到修订 run 中，以保留粗体、字号等格式。

---

## XML 参考

### Schema 合规性

- **`<w:pPr>` 中元素的顺序**：`<w:pStyle>`、`<w:numPr>`、`<w:spacing>`、`<w:ind>`、`<w:jc>`，最后才是 `<w:rPr>`
- **空白字符**：如果 `<w:t>` 含有前导或尾随空格，要添加 `xml:space="preserve"`
- **RSID**：必须是 8 位十六进制（例如 `00AB1234`）

### 修订痕迹

**插入：**
```xml
<w:ins w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:t>插入的文本</w:t></w:r>
</w:ins>
```

**删除：**
```xml
<w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>删除的文本</w:delText></w:r>
</w:del>
```

**在 `<w:del>` 内部**：使用 `<w:delText>`，不要使用 `<w:t>`；字段指令文本要使用 `<w:delInstrText>`，不要使用 `<w:instrText>`。

**最小化编辑** - 只标记真正变更的部分：
```xml
<!-- 将 “30 days” 改成 “60 days” -->
<w:r><w:t>The term is </w:t></w:r>
<w:del w:id="1" w:author="Claude" w:date="...">
  <w:r><w:delText>30</w:delText></w:r>
</w:del>
<w:ins w:id="2" w:author="Claude" w:date="...">
  <w:r><w:t>60</w:t></w:r>
</w:ins>
<w:r><w:t> days.</w:t></w:r>
```

**删除整段/整个列表项** - 当要删除一个段落中的全部内容时，还要把段落标记本身也标记为删除，这样接受修订后它才会与下一段合并。做法是在 `<w:pPr><w:rPr>` 内添加 `<w:del/>`：
```xml
<w:p>
  <w:pPr>
    <w:numPr>...</w:numPr>  <!-- 如果有列表编号，要保留 -->
    <w:rPr>
      <w:del w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z"/>
    </w:rPr>
  </w:pPr>
  <w:del w:id="2" w:author="Claude" w:date="2025-01-01T00:00:00Z">
    <w:r><w:delText>被删除的整段内容……</w:delText></w:r>
  </w:del>
</w:p>
```
如果缺少 `<w:pPr><w:rPr>` 里的 `<w:del/>`，接受修订后会留下一个空段落或空列表项。

**拒绝另一位作者的插入** - 把删除嵌套在对方的插入内部：
```xml
<w:ins w:author="Jane" w:id="5">
  <w:del w:author="Claude" w:id="10">
    <w:r><w:delText>对方插入的文本</w:delText></w:r>
  </w:del>
</w:ins>
```

**恢复另一位作者的删除** - 在其删除之后追加插入（不要直接修改对方的删除）：
```xml
<w:del w:author="Jane" w:id="5">
  <w:r><w:delText>被删除的文本</w:delText></w:r>
</w:del>
<w:ins w:author="Claude" w:id="10">
  <w:r><w:t>被删除的文本</w:t></w:r>
</w:ins>
```

### 批注

运行完 `comment.py` 之后（见第 2 步），需要把标记插入到 document.xml 中。若是回复批注，使用 `--parent` 参数，并把回复标记嵌套在父批注内部。

**关键：`<w:commentRangeStart>` 和 `<w:commentRangeEnd>` 必须与 `<w:r>` 同级，绝不能放在 `<w:r>` 内部。**

```xml
<!-- 批注标记必须是 w:p 的直接子节点，不能放进 w:r 内 -->
<w:commentRangeStart w:id="0"/>
<w:del w:id="1" w:author="Claude" w:date="2025-01-01T00:00:00Z">
  <w:r><w:delText>deleted</w:delText></w:r>
</w:del>
<w:r><w:t> more text</w:t></w:r>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>

<!-- 批注 0 内部嵌套回复 1 -->
<w:commentRangeStart w:id="0"/>
  <w:commentRangeStart w:id="1"/>
  <w:r><w:t>text</w:t></w:r>
  <w:commentRangeEnd w:id="1"/>
<w:commentRangeEnd w:id="0"/>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="0"/></w:r>
<w:r><w:rPr><w:rStyle w:val="CommentReference"/></w:rPr><w:commentReference w:id="1"/></w:r>
```

### 图片

1. 将图片文件放到 `word/media/`
2. 在 `word/_rels/document.xml.rels` 中添加关系：
```xml
<Relationship Id="rId5" Type=".../image" Target="media/image1.png"/>
```
3. 在 `[Content_Types].xml` 中添加内容类型：
```xml
<Default Extension="png" ContentType="image/png"/>
```
4. 在 document.xml 中引用该图片：
```xml
<w:drawing>
  <wp:inline>
    <wp:extent cx="914400" cy="914400"/>  <!-- EMU：914400 = 1 英寸 -->
    <a:graphic>
      <a:graphicData uri=".../picture">
        <pic:pic>
          <pic:blipFill><a:blip r:embed="rId5"/></pic:blipFill>
        </pic:pic>
      </a:graphicData>
    </a:graphic>
  </wp:inline>
</w:drawing>
```

---

## 依赖项

- **pandoc**：文本提取
- **docx**：`npm install -g docx`（用于新建文档）
- **LibreOffice**：PDF 转换（在受限环境中可通过 `scripts/office/soffice.py` 自动配置）
- **Poppler**：`pdftoppm`，用于导出图片
