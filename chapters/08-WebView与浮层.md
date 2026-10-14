# 08 · WebView 与浮层：跨平台混合合成

原生 WebView 无法进入引擎的 Vulkan 绘制流——Flutter 自己也做不到。项目用「多层原生表面 + 严格 z 序」近似：主 Vulkan 表面画引擎内容，原生 WebView 插在中间，透明第二表面画浮层。

## 8.1 窗口内 z 序：三层表面

1. **engineView** — 主 Vulkan 表面（不透明）——页面内容
2. **browser** — 原生容器：工具栏 + 状态行 + WebView
3. **widgetOverlay** — 透明 TextureView / CAMetalLayer / XComponent(TEXTURE) —— 第二块 Vulkan 表面，承载引擎浮层

第二表面使用透明清屏、预乘 Alpha 合成和独立命中判断，空白区域不获取触摸。三端结构统一，差别在验证程度：Android 有实战，iOS 与鸿蒙代码在但未真机验证。

## 8.2 基础接口：原生视图三件套

所有调用和回调都在 UI 线程。页面 State 持有 NativeViewHandle——句柄不可复制只可移动，**析构或 close() 会关闭对应原生视图**；不要留作临时变量，否则会立即析构关闭。

**State 成员 + 打开网页 / 弹窗 / 双槽 Toast**

```cpp
evk::ui::NativeViewHandle web;      // State 成员
evk::ui::NativeViewHandle dialog;
evk::ui::NativeViewHandle topToast;

evk::ui::WebViewOptions options;
options.title = "帮助";
options.url = "https://example.com";
// url 和 html 必须且只能指定一个；JS 默认关闭
web = evk::ui::showWebView(std::move(options));

evk::ui::ConfirmDialogOptions confirmation;
confirmation.title = "确认操作";
confirmation.message = "确定继续？";
dialog = evk::ui::showConfirmDialog(std::move(confirmation),
    [this](evk::ui::NativeViewEvent result) {
        if (result == evk::ui::NativeViewEvent::Confirmed) {
            // 执行业务操作
        }
    });

evk::ui::ToastOptions top;
top.message = "资讯已更新";
top.placement = evk::ui::ToastPlacement::Top;
top.offset = evk::ui::safeAreaInsets().top;
topToast = evk::ui::showToast(std::move(top));
```

- **insets 留白模型**：四边留白（引擎像素）指定原生容器显示区域，留白处的引擎页面透出——底部固定导航栏在网页打开期间保持可见，留白处的触摸仍归引擎。insets.bottom = 导航栏高 + safeAreaInsets().bottom；
- Toast 顶部/底部两个独立槽位，同位置新提示替换旧提示，默认 2 秒消失；
- 原生事件经 nativeViews/event 按会话 id 路由，再经 postUi 在 UI 帧交付；终结事件去重，显式关闭不再回调业务；
- **返回优先级**：handleNativeViewBack() 先处理弹窗 → 网页历史 → 关闭浏览页 → 框架焦点与页面导航。

## 8.3 Widget 浮层：完整 Widget 子树作为浮层

overlay.h 提供应用级浮层树，挂在 Navigator 根视图上，位于模态弹窗与遮罩之上。浮层内容是完整 Widget 子树——Text、图标、布局、按钮、自绘、StatefulWidget 都能用，不再局限于原生确认框的固定字段。

**模态弹窗 + 到期淡出的底部提示**

```cpp
evk::ui::OverlayOptions options;
options.modal = true;
options.maxWidth = 720;
options.barrierColor = 0x00000088;
customDialog = evk::ui::showOverlayWidget(
    evk::ui::sizedBox(600, 180, evk::ui::container(0xFFF0B8FF)), options);

evk::ui::OverlayOptions toast;
toast.placement = evk::ui::OverlayPlacement::Bottom;
toast.pointerTransparent = true;
toast.durationMs = 2000;
toast.fadeOutMs = 240;
customToast = evk::ui::showOverlayWidget(
    evk::ui::sizedBox(400, 80, evk::ui::container(0x1769C2FF)), toast);
```

> **⚠️ 注意事项**：浮层作为独立 Element 根挂载，不继承发起页面的 InheritedWidget——需要的主题等配置应显式传给浮层 Widget。App 返回事件应先调 handleOverlayBack()。模态期间框架焦点与无障碍语义限定在最上层弹窗。

## 8.4 WebView 路由转场：网页作为 Navigator 页面

Android、iOS、鸿蒙可将网页作为 Navigator 的页面，参与普通页面相同的 280 毫秒横向 push/pop 转场。Navigator 在布局之后、绘制之前同步网页容器的屏幕坐标、裁剪区域、可见性和交互状态；逐帧只平移原生容器，尺寸变化才重新布局，不重新加载网页。

**能力探测 + 路由化推入**

```cpp
if (evk::ui::supportsWebViewPages()) {
    evk::ui::WebViewOptions options;
    options.title = "资讯";
    options.url = "https://news.epolestar.xyz/notice?isEstarX=1&theme=0";
    options.javaScriptEnabled = true;
    evk::ui::appNavigator()->push(
        evk::ui::makeWidget<evk::ui::WebViewPage>(std::move(options)));
}
// 能力为假时回退到原生资讯浮层（留白方案）
```

- 覆盖网页只隐藏原生容器，保留同一个 WebView、历史和滚动位置；
- 当前原生宿主只容纳一个浏览器，同一导航栈已有网页路由时推第二个返回 false；
- WebViewOptions::insets 不参与路由布局，显示区域由 Navigator 内容区决定（iOS 像素 ÷ nativeScale，鸿蒙经 px2vp）；
- 网页顶栏「返回」发出 CloseRequested，框架执行 pop，退出动画结束后才释放网页。

## 8.5 平台视图化评估：WebView 能不能变成 widget？

结论：**可行，且比预想的近**——项目里已有半套 platform view 机制（nativeAnchor 前后景分层 + 第二块透明表面），缺的是把「锚点」从整页推广到任意矩形、从单实例推广到多实例。但目标应是「受约束的 widget」，不是「真正的 widget」。

| 能力 | 可达性 | 说明 |
|---|---|---|
| W1 进布局树，尺寸由父约束决定 | ✅ 完全可达 | 纯 C++，第一期就能做 |
| W2 矩形裁剪 / 随父位移 | ✅ 可达 | 平台原语已有（setClipBounds + translation） |
| W3 与引擎 Widget 任意叠放 | ⚠️ 有条件 | 靠多层表面；层数 = 交错段数 |
| W4 多实例、生命周期对齐 | ⚠️ 需要实例池 | 与「多 WebView 实例池」专项合并 |
| W5 随外层滚动一起滚 | ⚠️ 有损 | 逐帧跟随有 1 帧延迟，快速滚动会漂移/撕裂 |
| W6 圆角 / 变换 / 透明度 / 滤镜 | ⚠️ 平台差异大 | iOS 明确不支持 ShaderMask/ColorFiltered |
| W7 与引擎手势同一竞技场 | ❌ 不可达 | 原生视图直接收平台触摸，引擎只有 interactive 开关 |

| 路线 | 思路 | 判定 |
|---|---|---|
| A 纹理化 | VirtualDisplay + SurfaceTexture / TLHC 式 draw 重定向，把 WebView 变成引擎纹理 | 仅 Android 可行；iOS 无公开 API（WKWebView 不能渲进 Metal 纹理），死路 |
| B 混合合成（推荐） | 锚点从「页」变「区域表」；新增 PlatformViewWidget 三件套；容器管理器 + 实例池 | 第一期固定两层（整块网页 + 上下各一层引擎内容），覆盖 95% 场景 |
| C API 包装（伪 widget） | 保留全屏原生容器，C++ 接口包成 Widget 形状，内部仍走 insets + frame | 不足以作为终态，适合作为第一阶段交付 |

> **⛔ 反面清单**：① 不要尝试把 WKWebView 纹理化；② 不要让原生容器逐帧跟随滚动内容（必然撕裂）；③ 不要把「任意叠放」放进第一期目标；④ 不要在共享 GPU 设备落地前上多层表面（现状每层全量快照复制，层数一多就崩）；⑤ 不要去掉 supportsWebViewPages() 能力探测——三端成熟度不同，应用层回退必须保留。

## 8.6 调用链速查

1. **C++** — native_views.cpp —— showWebView / showConfirmDialog / showToast / WebViewPage
2. **platform_channel** — 字符串方法名，同步，UI 线程；nativeViews/web/{show,frame,close,insets,routesSupported}
3. **原生宿主** — Android NativeViewsHost.java · iOS ios_native_views.mm · 鸿蒙 NativeViews.ets

平台能力调用走 MethodChannel 式平台通道：按方法名路由、同步返回、全部跑在 UI 线程、负载为 UTF-8 串（core 不引 JSON 库，序列化各端自理）。
