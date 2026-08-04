# [222] PDF 阅读与文档编辑左侧目录（大纲）侧边栏

## 相关文档
- [x_y.md](x_y.md) - 开发者文档模板
- [Format_ZH.md](Format_ZH.md) - C++ 代码格式化指南
- [1170.md](1170.md) - 侧边栏 UI 实现（任务一、二）
- [1171.md](1171.md) - PDF 目录跳转（任务三）
- [1172.md](1172.md) - 文档目录跳转（任务四）

## 任务与 PR 编号

| 任务 | 描述 | PR 编号 |
|------|------|---------|
| 任务一 | 完成 PDF 阅读器的树形目录结构左侧侧边栏 | 1170 |
| 任务二 | 在同一个框架下完成文档编辑器的树形目录结构左侧侧边栏 | 1170 |
| 任务三 | 点击目录跳转到 PDF 阅读器的对应位置 | 1171 |
| 任务四 | 点击目录跳转到文档编辑器的对应位置 | 1172 |

## 任务相关的代码文件
- `src/Plugins/Qt/qt_pdf_reader_widget.hpp` - PdfOutlineItem / OutlineItem 结构、outlineLoaded 信号、extractOutline 声明
- `src/Plugins/Qt/qt_pdf_reader_widget.cpp` - extractOutline / walkOutline 实现（fz_load_outline + fz_resolve_link）
- `src/Plugins/Qt/qt_pdf_outline_widget.hpp` - OutlineWidget 类定义（QDockWidget + QTreeWidget + QTimer）
- `src/Plugins/Qt/qt_pdf_outline_widget.cpp` - setOutline / loadDocumentOutline / outlineActivated / setLiveRefresh 实现
- `src/Plugins/Qt/qt_tm_widget.hpp` - pdfOutlineDock 成员声明
- `src/Plugins/Qt/qt_tm_widget.cpp` - dock 创建、信号连接、update_visibility 可见性控制、实时刷新开关
- `TeXmacs/progs/text/text-outline.scm` - document-outline / outline-go-to Scheme 模块
- `TeXmacs/progs/init-research.scm` - 新增 (use-modules (text text-outline)) 加载
- `tests/Plugins/Qt/qt_pdf_reader_widget_test.cpp` - outline 提取断言用例
- `TeXmacs/tests/PDF/quartus_manual_with_outline.pdf` - 新增带大纲的测试 fixture

## 如何测试

### PDF 大纲
1. 打开含书签的 PDF（如 `TeXmacs/tests/PDF/quartus_manual_with_outline.pdf`）
2. 验证左侧出现「目录」dock，树形展示大纲，多层缩进可折叠展开
3. 点击条目 → PDF 滚动到对应页（页码准确，不差一页）
4. 关闭 PDF 再打开同一 PDF → dock 仍显示（不清空缓存）
5. 打开无大纲的 PDF → dock 不显示

### 文档大纲
1. 打开含 section/subsection 的 .tm 文档
2. 验证左侧出现「目录」dock，显示章节结构
3. 点击章节 → 光标跳转到对应章节位置（与 focus-section-menu 行为一致）
4. 编辑文档（添加/删除 section）→ 大纲自动刷新

### 单元测试
```bash
xmake b qt_pdf_reader_widget_test && xmake r qt_pdf_reader_widget_test
```

## 如何提交
```bash
# 格式化变更的 C++ 文件
gf fmt --changed-since=main
# 运行单元测试
xmake b qt_pdf_reader_widget_test && xmake r qt_pdf_reader_widget_test
```

## 2026/08/04 修复：跳转逻辑 + 实时同步

### What
修复了文档大纲跳转错误（跳到叶子节点而非节点本身），并新增编辑器模式下大纲实时同步。

### Why
- 跳转逻辑使用 `:start` 会定位到节点第一个字符，与现有 focus-section-menu 的 `:end` 行为不一致
- 编辑文档时章节结构会变化，大纲需要自动更新

### How
- 跳转逻辑改为 `(tree-go-to t 0 :end)`，与 `focus-section-menu`（`text-menu.scm:1029`）完全一致
- OutlineWidget 新增 QTimer（500ms 周期），编辑器模式下自动调用 loadDocumentOutline 刷新
- `setLiveRefresh(true/false)` 控制刷新开关：进入编辑器模式时开启，离开时关闭

## 2026/08/04 PDF 大纲 + 文档大纲

### What
在 PDF 阅读器和文档编辑器中新增左侧目录（大纲）侧边栏，支持点击跳转。

### Why
用户阅读 PDF 或编辑文档时需要结构化的导航能力。MuPDF 已提供 fz_load_outline 提取大纲树，项目已有 leftTools/chatSideDock 等 dock 范例可复用。

### How
- PDF 大纲：PDFReaderWidget 新增 extractOutline()（仿 extractPageLinks 的「重开 doc → fz_resolve_link」模式），loadFromFile 成功后 emit outlineLoaded 信号；OutlineWidget 接收 QVector<PdfOutlineItem> 建树；点击时 outlineActivated(target) 信号经 lambda 转为 goToPage(page+1)（0-based→1-based 修正）
- 文档大纲：新增 text-outline.scm 模块，document-outline 函数通过 tree-search-sections 获取章节列表并返回 (title path-str) 对；outline-go-to 解析路径字符串后调用 tree-go-to 跳转；OutlineWidget::loadDocumentOutline 通过 eval_scheme 获取数据并建树
- 可见性：update_visibility 在 PDF 模式按 hasContent() 显示 dock，编辑器模式同理；离开 PDF 模式时只 hide 不清空树内容，避免重复打开同一 PDF 时丢失大纲
