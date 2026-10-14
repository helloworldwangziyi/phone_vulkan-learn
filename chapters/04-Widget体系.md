# 04 · Widget 体系：机制 + home_page.cpp 案例走读

本章分两半：前半是 `widget_tree` 的机制（唯一对比规则、脏队列、生命周期），后半对着项目里最真实的页面——`samples/app/home_page.cpp`（617 行）逐段走读，看这些机制在真实代码里怎么用。

## 4.1 四类 Widget / Element 各司其职

| Widget 基类 | Element 实现 | 产生 View？ | rebuild 做什么 |
|---|---|---|---|
| `StatelessWidget` | StatelessElement | 否，透传孩子 | 重新 build()，交给 updateChild 继续对比 |
| `StatefulWidget` | StatefulElement | 否，透传孩子 | 调 State::build()，其余同上 |
| `RenderObjectWidget` | RenderObjectElement | **是，唯一直接持有 View** | updateRenderObject 刷参数 + 逐子对比 |
| `ProxyWidget` | ProxyElement | 否，只转交 | std::move(config.child()) 转交，改排布参数（Expanded 写 flex 系数） |

不产生 View 的三类，renderObject() 都是 `child_ ? child_->renderObject() : nullptr`——View 沿 Element 树向下透传，直到最近一个 RenderObjectElement。

## 4.2 唯一对比规则：updateChild 源码走读

`widget_tree.cpp:667`，整个框架的「守恒律」，没有任何一处特判：

1. **新 Widget 为空** — 旧子树 unmount 销毁，槽位置空（导航栈里的页面根返回 nullptr 时，会补一个 EmptyRouteWidget 顶上）
2. **canUpdate 为真（三重与）** — Key 相等 && typeid 动态类型相同 && 自定义 canUpdate → child->update()：换 Widget 副本，就地微调，State/View/滚动偏移/按压态全部保留
3. **其余（类型变了 / 槽位空）** — 旧子树 unmount → createElement → mount 重建；新 View 接管旧 View 的兄弟槽位（快照旧顺序 → reorderChildren 恢复）

> **ℹ️ 滚动位置保留、列表增删、主题换色不重建——全是这条规则的自然推论**：TextWidget 覆写 canUpdate 来区分单行/换行两种渲染对象（模式切换必须重建）。一切都是「微调」或「重建」二选一，没有第三条路。

## 4.3 生命周期：mount → update → rebuild → unmount

- **mount**（:571）：绑定挂靠信息 → firstMount（建树）→ flushLayout。StatefulElement 首次顺序固定：`createState() → attach → initState → rebuild() → didMount()`
- **update**（:588）：`willUpdateWidget` 是唯一能同时看到新旧 Widget 的窗口 → 换副本 → updateElement → flushLayout
- **rebuild**：四类各写各的；开头 beginBuild() 清脏——build 期间新来的 setState 排到下一帧，不递归重入
- **unmount**：顺序铁律**先拆孩子，再拆自身**（StatefulElement: 递归拆子树 → dispose() → detach() 清订阅 → 销毁 State；RenderObjectElement: 拆完子 Element 才把 View 从父 View 摘下）

> **💡 didMount 才注册事件监听**：didMount 在 View 已挂树之后才触发（State::listen 要拿 renderObject() 当订阅作用域），注册事件监听必须放这里而不是 initState。

## 4.4 帧头合帧：脏队列的两个巧设计

- **depth 排序 + buildDirty_ 去重 = 祖先覆盖孩子**：脏队列 stable_sort 按树深度升序，祖先先重建；被祖先递归覆盖的孩子轮到它时 buildDirty_ 已清，直接跳过。回归测试：100 次 setState 只产生 1 次 build（tests/incremental_ui_test.cpp:61）
- **lifetime 弱令牌 = 免手动出队**：队列存 weak_ptr&lt;uint8_t&gt;，unmount 时 reset，残留条目 flush 时 expired() 一跳而过。测试场景很刁钻：先 setState 排队，再让祖先把它删掉（:81）
- **建树期间不布局**：g_treeUpdateDepth / g_flushingBuilds 两个全局守卫，View::flushLayout 开头唯一消费点——半建成的树不重排

## 4.5 案例走读：home_page.cpp 的四个状态组件

HomePage 的 build() 有 15 个子节点，但页面自身几乎没有可变状态——全部拆给了四个独立的 StatefulWidget。这是「状态下沉」范式的完整范本。

### 案例 1 · HomeMarketStatus：listen + didMount + onDidEnter 三件套

**samples/app/home_page.cpp:106-126**

```cpp
class HomeMarketStatusState final : public evk::ui::State {
    void initState() override { status_ = marketStatusText(*appMarketState()); }
    void didMount() override {
        listen(kEventMarketData, evk::ui::EventPriority::Normal,
               [this](const void*) { refresh(); });
    }
    void onDidEnter(bool) override { refresh(); }   // 回前台补齐覆盖期间错过的事件
    std::unique_ptr<evk::ui::Widget> build(...) override {
        return evk::ui::text(status_, ...);
    }
private:
    void refresh() {
        auto next = marketStatusText(*appMarketState());
        if (next == status_) return;   // ← 有变化才 setState
        status_ = std::move(next);
        setState();
    }
};
```

三个钩子各司其职：initState 给初值（还不用 View）、didMount 注册订阅（此时 View 已挂树）、onDidEnter 在页面被覆盖又回来时主动刷新——因为 listen 的 scope 是页面根 View，被覆盖期间事件**隐式暂停**。refresh() 返回前判等，**没变化不 setState**。

### 案例 2 · HomeQuoteRow + HomeQuoteModel：行级精确重建

**samples/app/home_page.cpp:144-192（节选）**

```cpp
struct HomeQuoteModel {
    HomeQuote value;
    std::function<void()> changed;
    void update(const evk::CspQuote& quote) {
        const auto change = marketChange(quote);
        HomeQuote next{ ... , quote.stale, !change || *change >= 0};
        if (value == next) return;     // ← 可见字段没变，什么都不发生
        value = std::move(next);
        if (changed) changed();        // ← 通知行 State：只有数据变了的那行
    }
};

class HomeQuoteRowState final : public evk::ui::State {
    void dispose() override { if (model_) model_->changed = {}; }  // 断开回调
    std::unique_ptr<evk::ui::Widget> build(...) override {
        const auto& config = widgetAs<HomeQuoteRow>();
        if (model_ != config.model) {           // Widget 配置换了模型 → 换绑回调
            if (model_) model_->changed = {};
            model_ = config.model;
            model_->changed = [this] { setState(); };
        }
        ...
        const auto color = quote.stale ? theme.textSecondary
                          : quote.up ? 0xEF5350FF : 0x26A69AFF;   // 红涨绿跌
        return padding(..., column(widgetList(
            text(quote.title, ...), text(quote.price, color, ...))));
    }
};
```

设计要点：model 用 `shared_ptr` 存在 State 外（HomeMarketList 持有），行 State 通过 changed 回调反压。dispose 里把回调置空——页面销毁后 model 即使还在被更新也不会碰已死的 State。一次行情广播，**只有数值变了的行触发 setState**，其余行走 reconcile 时空操作。

### 案例 3 · HomeMarketList：keyed 保住行身份

**samples/app/home_page.cpp:212-243（节选）**

```cpp
std::unique_ptr<evk::ui::Widget> build(...) override {
    std::vector<std::unique_ptr<Widget>> rows;
    for (const auto& contract : contracts_) {
        rows.push_back(keyed(Key(contract),          // ← Key = 业务主键
            makeWidget<HomeQuoteRow>(models_.at(contract))));
    }
    return scrollView(column(std::move(rows)));
}

bool refresh() {
    const auto market = appMarketState();
    for (const auto& quote : market->quotes) {
        auto& model = models_[quote.contract];
        if (!model) model = std::make_shared<HomeQuoteModel>();
        model->update(quote);                      // 报价更新：不动列表结构
        contracts.push_back(quote.contract);
    }
    if (contracts == contracts_) return false;     // 增删/换序才重建列表
    ...
}
```

> **ℹ️ 两层粒度的配合**：refresh() 里 `model->update()` 先跑——报价变化在**行级**解决（案例 2 的 changed 回调）；只有合约增删/换序才返回 true 触发整表 setState，此时 keyed(Key(contract)) 让每行按业务主键对齐复用，行 State 不丢。

### 案例 4 · InheritedThemeDemo：与 EventBus 对照的精确重建

**samples/app/home_page.cpp:44-98（节选）**

```cpp
// StatelessWidget 在 build 里登记依赖：作用域换色时只重建这个盒子
std::unique_ptr<evk::ui::Widget> build(evk::ui::BuildContext& context) const override {
    const ThemeScope& scope = ThemeScope::of(context);   // ← dependOn 登记点
    return evk::ui::container(scope.boxColor(), onTap_, ...);
}

// State 持有 dark 标志，ThemeScope 把它沿树下发
std::unique_ptr<evk::ui::Widget> build(evk::ui::BuildContext&) override {
    return evk::ui::makeWidget<ThemeScope>(dark_,
        evk::ui::makeWidget<ScopedThemeBox>([this] {
            setState([this] { dark_ = !dark_; });
        }));
}
```

对照 home_page 顶上的主题切换按钮：那边是 `EventBus::emit(kEventThemeChanged)` → 每个页面的 didMount 监听 → 整页 setState；这里是 InheritedWidget → 只有 build 里调用 `ThemeScope::of(context)` 登记过的子树重建。两条路径在同一个文件里，注释写明了是刻意的对照演示。

### 案例 5 · panel painter 的外部数据陷阱

**samples/app/home_page.cpp:267-294（节选）**

```cpp
auto panel = padding(..., sizedBox(-1.0f, appCalcHeight(540.0f),
    container(theme.surface, [this] {
        setState([this] { panelAccent_ = !panelAccent_; });   // ← 必须 setState
    }, [accent = panelAccent_](PaintContext& paint) {         // ← 按值捕获!
        const AppTheme& currentTheme = appTheme();            // ← 主题现场重读
        paint.drawTriangle(..., accent ? currentTheme.panelAccent
                                       : currentTheme.panelGradient[0], ...);
    })));
```

> **⚠️ painter 的三条纪律（这个 lambda 全占了）**：① **按值捕获**外部数据（accent = panelAccent_）——painter 在绘制期执行，引用捕获可能读到已变的值；② 主题这类全局 token **绘制时现场重读**——深/浅切换后 painter 不重录，读旧值就画错色；③ 点击里必须走 setState——直接改成员变量不会触发重绘（除非你手动 view.markNeedsPaint()）。

### 案例 6 · 平台通道 + 导航 push 的真实写法

**samples/app/home_page.cpp:296-301 与 :486-493**

```cpp
// push 带参页面：makeWidget<DetailPage>(detailCount_++)，参数进 Widget 构造函数
context().navigator().push(makeWidget<DetailPage>(detailCount_++), true);

// 出向平台调用：同步、UI 线程、失败有兜底文案
std::string info;
const bool ok = evk::invokePlatform("deviceInfo", "", &info);
setState([this, ok, info = std::move(info)] {
    platformInfo_ = ok && !info.empty() ? info : "平台未实现 deviceInfo";
});
```

## 4.6 State 契约速查 + 两条纪律

| 接口 | 语义 |
|---|---|
| `initState()` | 首次 build 之前调用一次（还拿不到 View） |
| `didMount()` | View 已挂树后调用一次——注册 listen 的地方 |
| `dispose()` | 子树销毁前调用——断开回调/取消请求的最后时机（案例 2） |
| `build()` | 唯一必须实现的方法，必须便宜（读快照可以，发请求不行） |
| `setState(mutate)` | mutate 立即执行，build 排到帧头，一帧合并 |
| `listen(id, prio, fn)` | 订阅绑页面根 View，被覆盖时隐式 pause，detach 自动退订 |
| 导航钩子 | onWillEnter / onDidEnter / onWillLeave（返回 false 否决）/ onDidLeave（案例 1 用它补齐数据） |

> **⚠️ 两条纪律**：① setState 里的 mounted() 检查是迟到数据的第一道防线；② listen 的 scope 是页面根 View——被覆盖时事件不送达，回前台要自行重读补齐（案例 1 的 onDidEnter 就是这个问题的标准答案）。
