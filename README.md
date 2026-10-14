# estarx_vulkan 学习指南

一个用 **C++17 + Vulkan** 写成的跨平台 UI 框架。

estarx_vulkan 是一套小型跨平台 C++17 UI 框架，由 Vulkan 渲染，API 全面参照 Flutter 的保留式声明模型——但**全部是 C++**：没有 view 句柄、没有 C 回调、没有 user_data 指针，也没有任何公开 C ABI 层。本指南按「语言特性 → 渲染 → 线程 → UI 框架 → 平台混合」的路线，带你逐层读透这个项目。

`C++17` · `Vulkan` · `Flutter 式三棵树` · `Android / iOS / HarmonyOS` · `自绘 SDF 图形` · `CSP 行情实战`

## 目录

- [00 · 项目概览](chapters/00-项目概览.md)
- [01 · C++ 特性：语言底座](chapters/01-Cpp特性.md)
- [02 · 渲染：从 drawRect 到三角形上屏](chapters/02-渲染管线.md)
- [03 · 多线程：UI、Raster、Worker 与 CSP](chapters/03-多线程模型.md)
- [04 · Widget 体系 + home_page.cpp 走读](chapters/04-Widget体系.md)
- [05 · 布局：盒约束协议](chapters/05-布局约束.md)
- [06 · 滑动与手势：竞技场、ScrollView、拖拽](chapters/06-滑动与手势.md)
- [07 · 动画：逐帧 tick 驱动的世界](chapters/07-动画与转场.md)
- [08 · WebView 与浮层：跨平台混合合成](chapters/08-WebView与浮层.md)

> 本站内容为 estarx_vulkan 项目 `文档/` 目录的学习性整理，深入细节请对照源码与原始文档。
