# PDF 阅读左侧目录（大纲）侧边栏

## What

在 PDF 标签页模式(`pdfTabMode`)下，界面左侧展示 PDF 的目录（大纲/bookmark/outline），
点击条目跳转到对应页。

## Why

用户阅读 PDF 时需要目录导航能力，MuPDF 已提供 `fz_load_outline` 提取大纲树，
现有 `extractPageLinks` 已有「重开 doc → `fz_resolve_link` 解析目标页」的成熟模式可复用。

## How

1. `qt_pdf_reader_widget.{hpp,cpp}`：新增 `PdfOutlineItem` 结构，`extractOutline()` 用
   `fz_load_outline` + `fz_resolve_link` 提取大纲树，`loadFromFile` 成功后 emit
   `outlineLoaded` 信号。
2. `qt_pdf_outline_widget.{hpp,cpp}`（新文件）：`PdfOutlineWidget : QDockWidget`，
   内含 `QTreeWidget`，`setOutline` 递归建树，`itemClicked` 发 `pageRequested` 信号。
3. `qt_tm_widget.{hpp,cpp}`：新增 `pdfOutlineDock` 成员，仿 `chatSideDock` 挂载左侧；
   `pdfViewerWidget` 首次创建时连接 `outlineLoaded → setOutline`、`pageRequested → goToPage`；
   `update_visibility` 在 PDF 模式且目录非空时显示 dock；离开 PDF 模式时隐藏并清空。
4. `qt_pdf_reader_widget_test.cpp`：新增两个用例，用 `QSignalSpy` 监听
   `outlineLoaded`，分别验证无大纲 PDF 发出空 outline、有大纲 PDF 解析出有效条目。

## 涉及文件

- `src/Plugins/Qt/qt_pdf_reader_widget.hpp` — PdfOutlineItem 结构、outlineLoaded 信号、extractOutline 声明
- `src/Plugins/Qt/qt_pdf_reader_widget.cpp` — extractOutline 实现
- `src/Plugins/Qt/qt_pdf_outline_widget.hpp/.cpp` — 新文件，PdfOutlineWidget
- `src/Plugins/Qt/qt_tm_widget.hpp` — pdfOutlineDock 成员
- `src/Plugins/Qt/qt_tm_widget.cpp` — dock 创建/连接/可见性
- `tests/Plugins/Qt/qt_pdf_reader_widget_test.cpp` — outline 提取测试
- `TeXmacs/tests/PDF/quartus_manual_with_outline.pdf` — 新增测试 fixture
