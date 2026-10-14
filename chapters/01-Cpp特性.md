# 01 · C++ 特性：这套框架的语言底座

项目用 C++17 的对象语义替代了 Flutter（Dart）的引用语义 + 垃圾回收：所有权由 **std::unique_ptr** 显式表达，生命周期由 **RAII** 守卫，不可变性由**类型系统**强制。这一章是后面所有章节的前置知识。

## 1.1 纯 C++ 对象接口：没有 C ABI

框架对应用暴露的全是 C++ 抽象类与值类型：**没有** view 句柄、C 回调、user_data 指针或公开的 C 链接层。C 链接只出现在 JNI、Objective-C 和平台工厂这类操作系统边界。这意味着：

- 编译器替你检查生命周期——悬垂指针大多是编译期错误；
- 析构顺序即资源回收顺序，C++ 所有权直接完成资源回收；
- 跨线程的对象图永远是清晰的「树 + 明确的交接点」（见多线程一章）。

## 1.2 移动语义与编译期不可变

Widget 被设计成**只能移动、不能拷贝**——「要变就造新的」这条 Flutter 契约，在这里变成了编译期错误而不是靠自觉（widget_tree.h）：

**core/src/ui/widget_tree.h · Widget 的拷贝控制**

```cpp
Widget(const Widget&) = delete;
Widget& operator=(const Widget&) = delete;
Widget(Widget&&) = default;
```

全项目到处都是 `std::move`：新 Widget 树在 build() 里整体 move 出来，updateChild 时用 move 把配置「换岗」到常驻 Element 上（`widget_ = std::move(widget)`），ProxyElement 的 rebuild 只有四行——`std::move(config.child())` 转交即可。

## 1.3 RAII：一切「要记得清理」的东西都是对象

| RAII 对象 | 守卫的资源 | 释放时机 |
|---|---|---|
| `EventSubscription` | EventBus 订阅 | State 销毁 / detach 时自动退订 |
| `WorkerTask` | 线程池任务与未交付回调 | 析构 / 替换 / cancel() |
| `NativeViewHandle` | 原生 WebView / 弹窗 / Toast | 句柄析构或 close()，不可复制只可移动 |
| `OverlayHandle` | Widget 浮层子树 | close() / 页面销毁 |
| `TreeUpdateScope` | 建树期间的布局抑制 | 出右大括号析构，计数归零 |

> **💡 刻意写成大括号**：mount / update 里那对「看起来多余的大括号」是刻意的 RAII 作用域：`{ TreeUpdateScope scope; firstMount(); }`。计数 +1 期间 flushLayout 全被抑制，右大括号析构后那一次 flushLayout 才是正式布局。注释写得比代码长。

## 1.4 unique_ptr 所有权：三棵树的内存结构

三棵树全部由 unique_ptr 构成父子链：Element 持有孩子 Element，View 由 RenderObjectElement 独占持有并 `addChild` 挂到父 View。Element 卸载时按树顺序移除 View，析构自动沿链回收——没有 GC，也没有手动 delete。

**createRenderObject 在 mount 期恰好调用一次（widget_tree.cpp）**

```cpp
std::unique_ptr<View> created = config.createRenderObject();  // 虚调用
if (renderParent_) view_ = renderParent_->addChild(std::move(created));
else               detachedView_ = std::move(created);  // 页面根:暂存游离
updateChildren(config);                                  // 递归建孩子
```

## 1.5 weak_ptr 生命周期令牌：跨帧引用的安全网

脏队列里存的不是裸 Element 指针，而是 `weak_ptr<uint8_t>` 弱令牌：Element unmount 时 `lifetime_.reset()`，队列里的残留条目自动失效，flush 时 expired() 一跳而过——**免手动出队**。

同一套模式复用在 `ViewRef`（render_view.h）：裸地址 + 弱令牌，让输入分发和动画能安全检测目标 View 是否已销毁。测试场景很刁钻：先 setState 排队，再让祖先把它删掉。

## 1.6 RTTI：对比规则依赖 typeid / dynamic_cast

canUpdate 是三重与：Key 相等、动态类型相同（typeid）、自定义 canUpdate 钩子。全项目共 42 处 RTTI 使用，当前构建未关闭 RTTI——一旦引入 -fno-rtti，这套对比机制会直接失效（源码里坦承这个耦合暂无注释提示）。

**canUpdate：三重与（widget_tree.cpp）**

```cpp
return oldWidget->key == nextWidget.key
    && typeid(*oldWidget) == typeid(nextWidget)
    && oldWidget->canUpdate(nextWidget);
```

## 1.7 lambda、std::function 与 std::any

- **std::function**：EventBus 回调、动画帧回调（animation_scheduler 逐帧驱动）、painter lambda，全部走类型擦除的 std::function。
- **无捕获 lambda 转函数指针**：WorkerPool 的 work 必须是普通函数或无捕获 lambda，`+[]` 前缀把它转成函数指针，从类型上拒绝捕获 this。
- **std::any**：拖拽数据通道。Draggable 发起时复制 std::any，目标端 `std::any_cast<T>(&data)` 检查类型。
- **模板方法模式**：View::layout 是非虚骨架（幂等短路 → 存约束 → 虚调 performLayout → 钳制写尺寸 → 清脏），performLayout 是虚钩子——框架不变量收在基类，子类无法破坏。

**WorkerPool：+[] 无捕获 lambda 进入线程池**

```cpp
task_ = evk::WorkerPool::instance().compute(
    std::move(input),
    +[](std::vector<float> values) {   // + 将无捕获 lambda 转成函数指针
        for (auto& value : values) value *= 2;
        return values;
    },
    [this](std::vector<float> result) {  // 完成回调只在 UI 线程执行
        setState([&] { values_ = std::move(result); });
    },
    [this](std::exception_ptr error) { /* UI 上处理失败 */ });
if (!task_) { /* 等待队列已满，可稍后重试 */ }
```

> **ℹ️ 语言选择的一致性结论**：C++17 的 move-only 类型 + RAII + 弱令牌，恰好一一对应了声明式 UI 需要的「不可变描述、确定性销毁、安全异步」三件套——这是 Dart + GC 模型的另一种解法。
