# WinUI 3 通用对话页：布局、键盘（Enter / Shift+Enter）、自动滚动

## 文档用途

- 本文是 **WinUI 3** 下「经典对话页」的通用知识库：**上方标题/工具条、中间消息列表、底部输入区**；以及 **Enter 发送、Shift+Enter 换行**、**列表有条件自动滚到底**。
- 与 **AgentClient / 流式 AI SDK** 解耦：只约定「谁在何时调用 `ScrollHistoryToBottomAsync`」，不关心消息来自 AI、客服还是本地模拟。
- 目标：以后新做「包括但不限于 AI」的对话页，**照本文档落地**，不必再翻历史 `ChatPage` 照抄。

## 术语说明

| 术语 | 含义 |
|------|------|
| **有条件自动滚动** | 仅当用户当前停留在列表底部附近时，新内容追加才自动滚到底；用户上滑阅读历史时暂停跟随，回到底部附近后恢复。 |
| **`force` 滚动** | 忽略「是否在底部」判断，强制滚到底（典型：用户点击发送后，明确表达「要看最新」）。 |
| **`UpdateSourceTrigger`** | 文本绑定写回 ViewModel 的时机；输入过程中同步需 `PropertyChanged`。 |
| **`e.Handled`** | 标记该路由事件已由当前逻辑处理，抑制控件默认行为（如 Enter 插入换行）。 |

## 通用需求背景

- 布局：**顶栏（可选）** + **中间 `*` 消息区** + **底栏输入**。
- 输入框：**Enter 提交**、**Shift+Enter 换行**；长文本可多行显示（`MaxHeight` + 纵向滚动条）。
- 绑定：`TextBox` 默认常见为 **失焦才写回** `TwoWay` 源；若发送逻辑读 ViewModel 字符串，需在输入过程中同步，否则 **Enter 提交读到旧值**。
- 列表：`ListView` 内部自带 `ScrollViewer`，需 **在视觉树中查找** 后订阅 `ViewChanged`，用 **距底部阈值** 维护 `shouldAutoScroll`。
- 线程：流式或后台线程更新 UI 时，仍需在 **UI 线程** 改集合/文本并触发滚动；本文 Demo 用 `DispatcherQueue`；业务层自行封装即可。

## 布局基线（推荐）

### 采用 `Grid` 三行：`Auto` / `*` / `Auto`

- **第 0 行（`Height="Auto"`）**：标题、会话名、工具按钮等（不需要可删整行，中间区改为 `Grid.Row="0"`）。
- **第 1 行（`Height="*"`）**：消息列表，占满剩余空间。
- **第 2 行（`Height="Auto"`）**：输入栏（`TextBox` + `Button`），不被中间区挤压消失。

**原则**：中间历史区必须吃 `*`，底栏永远固定高度策略（`Auto` + 输入控件 `MaxHeight`），避免输入区被挤出可视区域。

### 中间区容器

- 可用 `Border` / `Frame` 包一层 `ListView`，统一圆角与描边。
- `ListView`：`SelectionMode="None"` 减少误选；气泡内 `TextBlock.IsTextSelectionEnabled="True"` 便于用户复制。

### 底栏输入

- `TextBox`：`TextWrapping="Wrap"`，`MaxHeight` + `ScrollViewer.VerticalScrollBarVisibility="Auto"` 控制多行上限与滚动。
- **不要**依赖 `AcceptsReturn="True"` 实现「Enter 发送」：为 `True` 时控件会先处理回车，`KeyDown` 里即使 `Handled` 仍易出现「既发送又换行」的竞态。基线：**不显式打开 `AcceptsReturn`**（默认 `False`），回车行为全部由 `KeyDown` 接管。
- 绑定：`Text="{x:Bind ViewModel.Draft, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"`，保证按键处理与发送读到的字符串一致。

## 键盘：`Enter` 发送、`Shift+Enter` 换行（项目内稳定写法）

### 检测 Shift

使用 `Microsoft.UI.Input.InputKeyboardSource.GetKeyStateForCurrentThread(VirtualKey.Shift)`，并用 **`HasFlag(CoreVirtualKeyStates.Down)`** 判断是否按下 Shift（与位运算等价，可读性更好）。

### 分支行为

- **`Enter` 且无 Shift**：执行发送命令（如 `SendCommand.Execute(null)`），并 **`e.Handled = true`**，避免 `TextBox` 默认处理。
- **`Enter` 且 Shift**：在 ViewModel 的草稿末尾追加 **`Environment.NewLine`**，将 `TextBox.SelectionStart` 设为 **`Draft.Length`**，把光标放到文末；**`e.Handled = true`**。

### 设计取舍（务必知晓）

- 当前写法是 **始终在文本末尾追加换行**并把光标放到 **文末**。适合「从下到上连续输入」的聊天草稿；若产品要求 **在光标中间插入换行**，需改为读取 `SelectionStart` / `SelectionLength` 后拼接字符串（与 MAUI 文档里「中间插入」思路一致），**不要**假设 `UserInput +=` 一种写法通吃所有场景。

### `e.Handled` 的作用（摘要）

- **`true`**：本次按键视为已由你处理，抑制控件默认行为（尤其 Enter）。
- **整段 `Enter` 分支末尾统一 `Handled = true`**：避免 Shift 与非 Shift 路径遗漏。

## 自动滚动：状态机 + `ScrollHistoryToBottomAsync`

### 状态变量

```csharp
const double AutoScrollBottomThreshold = 24; // 距底部小于等于此值视为「在底部」，可按 DPI 调 16~48
bool shouldAutoScroll = true;
ScrollViewer? listScrollViewer;
```

### 取得 `ListView` 内部的 `ScrollViewer`

`ListView` 不直接暴露滚动偏移，需在 **`Loaded`** 后对命名 `ListView` 做 **视觉树 DFS**，找到第一个 `ScrollViewer` 并缓存；在 **`Unloaded`** 里取消 `ViewChanged` 订阅并置空，避免泄漏与重复订阅。

### `ViewChanged` 中更新 `shouldAutoScroll`

```csharp
var scrollable = sv.ScrollableHeight;
if (double.IsNaN(scrollable) || double.IsInfinity(scrollable)) return;
var distanceToBottom = scrollable - sv.VerticalOffset;
shouldAutoScroll = distanceToBottom <= AutoScrollBottomThreshold;
```

### `ScrollHistoryToBottomAsync(bool animated = true, bool force = false)`

1. 若 **`!force && !shouldAutoScroll`**：直接返回（用户正在看历史）。
2. 若当前不在 UI 线程：用 **`DispatcherQueue.TryEnqueue`** 切回 UI 线程再滚（与 MAUI `MainThread` 同理）。
3. 滚动前 **`Task.Delay(16)`** 量级：给布局一次测量机会，减少「内容已变但 `ScrollableHeight` 尚未更新」导致的滚底失败。
4. 使用 `ScrollViewer.ChangeView(null, y, null, disableAnimation: !animated)`，`y = ScrollableHeight`。

### 谁应该调用滚底？

| 场景 | 建议调用 |
|------|-----------|
| 模拟/真实 **流式** 追加正文（用户未上滑） | `ScrollHistoryToBottomAsync(animated: false, force: false)` |
| **流式结束**、或一次插入大块文本 | `ScrollHistoryToBottomAsync(animated: true, force: false)` 或保持 `false` 依动画需求 |
| **用户点击发送**后 | **`force: true`**，强制回到底部看最新 |

**禁止**：对流式每一段都用 `force: true`，否则用户上滑阅读时仍会被拽回底部。

## 与业务（含 AI）解耦的对接方式

只保留一个 **UI 侧契约** 即可，二选一或组合：

### A. 接口注入（推荐测试与复用）

```csharp
public interface IConversationScrollTarget
{
    Task ScrollHistoryToBottomAsync(bool animated = true, bool force = false);
}
```

页面实现该接口；ViewModel 或中间层只依赖接口，不关心页面是否接 AI。

### B. 事件 / 消息总线

例如 `ConversationMessageAppended`（附带 `bool isUserInitiated`）：`isUserInitiated == true` 时 UI 用 `force: true` 滚底；否则按 `shouldAutoScroll` 规则调用。

**AI 特有逻辑**（工具调用、Chunk 解析、`FinishReason`）全部放在 **Hosting / Service** 层，对话页只接收「最终要显示的字符串」或「已构造好的消息项」。

## 关键原则（必须遵守）

1. **`TextBox` 发送前绑定必须已更新**：`UpdateSourceTrigger=PropertyChanged` 或在发送前从控件读 `Text`。
2. **Enter 发送不要用 `AcceptsReturn=True` 偷懒**：与 `KeyDown` 组合易产生双重行为。
3. **自动滚动必须「有条件 + 强制」两套路径**：对流式用条件，对用户发送用强制。
4. **`ScrollViewer` 查找结果缓存 + `Unloaded` 解绑**，避免重复事件与泄漏。
5. **非 UI 线程更新集合**：先切 `DispatcherQueue`，再改 `ObservableCollection` 与滚底。

## WinUI 补充：`NavigationCacheMode`（可选）

若该页放在 `Frame` 中导航，且希望 **离开再返回仍保留输入草稿与列表状态**，在 `Page` 上设 `NavigationCacheMode="Required"`。这与滚底、键盘逻辑正交；按需开启。

## 常见问题与处理

| 现象 | 可能原因 | 处理 |
|------|-----------|------|
| 按 Enter 发送的是旧内容 | 绑定更新在 `LostFocus` | `UpdateSourceTrigger=PropertyChanged` |
| Enter 仍换行 | `AcceptsReturn=True` 或未 `e.Handled` | 默认不开启 `AcceptsReturn`；Enter 分支 `Handled=true` |
| Shift+Enter 不换行 | 未改 ViewModel 或未同步到 `TextBox` | 改绑定属性并设 `SelectionStart` |
| 流式不滚底 | 未调用滚底或 `shouldAutoScroll=false` | 追加内容后调用；用户发送用 `force` |
| 上滑仍被拽回底 | 流式路径误用 `force: true` | 流式仅用 `force: false` |
| `ScrollableHeight` 为 0 | 布局未完成 | 保留 `Delay(16)` 或监听 `LayoutUpdated`（进阶） |
| 找不到 `ScrollViewer` | `Loaded` 过早或模板不同 | 延后到 `Loaded`、或换 `ItemsRepeater+ScrollViewer` 显式结构 |

## 最小可移植清单（跨项目复制时）

1. 复制 **`Grid` 三行布局**（顶/中/底）骨架。
2. 复制 **`FindDescendantScrollViewer` + `Loaded`/`Unloaded` + `ViewChanged` + `shouldAutoScroll`**。
3. 复制 **`ScrollHistoryToBottomAsync` + `DispatcherQueue` 分支**。
4. 复制 **`TextBox` 绑定 `PropertyChanged` + `nameUserInput_KeyDown` 模板**（按产品决定是否改为光标处插换行）。
5. 将「消息追加」处接上 **条件滚底**；将「用户发送」处接上 **强制滚底**。
6. 把 `DataTemplate`、消息模型、发送命令替换为目标项目实现。

---

## 附录 A：与 AI / Agent 相关的替换表（落地前映射）

| 知识库 Demo / 占位 | 替换为 |
|--------------------|--------|
| `DemoChatMessage` | 你的消息 DTO / 领域模型 |
| `SendCommand` | 你的发送入口（HTTP、SignalR、本地队列等） |
| `SimulateStreamAppendAsync` | `OnStreamOutput` / `IAsyncEnumerable` 消费处 |
| `IConversationScrollTarget` | 或你团队统一的 `INavigationService` / `IMessenger` |

---

## 附录 B：完整通用 Demo（无 AI、无 AgentClient）

> 以下代码为 **自洽示例**：单文件 ViewModel + Page，仅依赖 **WinUI 3** 与 **CommunityToolkit.Mvvm**（`ObservableObject`、`RelayCommand`）。  
> 复制到新 WinUI 项目时：改命名空间、XAML `x:Class`、将 `Page` 挂到 `Window`/`Frame` 即可运行。

### B.1 `GenericConversationDemoPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Page
    x:Class="YourApp.Pages.GenericConversationDemoPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:local="using:YourApp.Pages"
    x:DataType="local:GenericConversationDemoViewModel"
    NavigationCacheMode="Required"
    mc:Ignorable="d">

    <Grid RowSpacing="10" Padding="12">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
            <RowDefinition Height="Auto" />
        </Grid.RowDefinitions>

        <TextBlock
            Grid.Row="0"
            Text="{x:Bind ViewModel.SessionTitle, Mode=OneWay}"
            HorizontalAlignment="Center"
            FontWeight="SemiBold" />

        <Border
            Grid.Row="1"
            BorderBrush="{ThemeResource CardStrokeColorDefaultBrush}"
            BorderThickness="1"
            CornerRadius="8"
            Padding="8">
            <ListView
                x:Name="nameMessageList"
                ItemsSource="{x:Bind ViewModel.Messages, Mode=OneWay}"
                SelectionMode="None">
                <ListView.ItemTemplate>
                    <DataTemplate x:DataType="local:DemoChatMessage">
                        <StackPanel
                            Margin="0,0,0,8"
                            Padding="8"
                            Spacing="4"
                            BorderBrush="{ThemeResource CardStrokeColorDefaultBrush}"
                            BorderThickness="1"
                            CornerRadius="8">
                            <TextBlock FontWeight="SemiBold" Text="{x:Bind Sender, Mode=OneWay}" />
                            <TextBlock Text="{x:Bind Body, Mode=OneWay}" TextWrapping="Wrap" IsTextSelectionEnabled="True" />
                        </StackPanel>
                    </DataTemplate>
                </ListView.ItemTemplate>
            </ListView>
        </Border>

        <Grid Grid.Row="2" ColumnSpacing="10">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*" />
                <ColumnDefinition Width="Auto" />
            </Grid.ColumnDefinitions>

            <TextBox
                x:Name="nameDraftInput"
                Grid.Column="0"
                PlaceholderText="Enter 发送，Shift+Enter 换行"
                Text="{x:Bind ViewModel.Draft, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"
                TextWrapping="Wrap"
                MaxHeight="200"
                ScrollViewer.VerticalScrollBarVisibility="Auto"
                KeyDown="nameDraftInput_KeyDown" />

            <Button
                Grid.Column="1"
                Content="发送"
                Style="{StaticResource AccentButtonStyle}"
                Command="{x:Bind ViewModel.SendCommand}" />
        </Grid>
    </Grid>
</Page>
```

### B.2 `GenericConversationDemoPage.xaml.cs`

```csharp
using System;
using System.Collections.ObjectModel;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.UI.Dispatching;
using Microsoft.UI.Input;
using Microsoft.UI.Xaml;
using Microsoft.UI.Xaml.Controls;
using Microsoft.UI.Xaml.Input;
using Microsoft.UI.Xaml.Media;
using Windows.System;
using Windows.UI.Core;

namespace YourApp.Pages;

public sealed partial class GenericConversationDemoPage : Page, IConversationScrollTarget
{
    const double AutoScrollBottomThreshold = 24;

    ScrollViewer? _listScrollViewer;
    bool _shouldAutoScroll = true;

    public GenericConversationDemoViewModel ViewModel { get; private set; } = null!;

    public GenericConversationDemoPage()
    {
        InitializeComponent();
        ViewModel = new GenericConversationDemoViewModel(this, DispatcherQueue);
        DataContext = ViewModel;
        Loaded += OnLoaded;
        Unloaded += OnUnloaded;
    }

    void OnLoaded(object sender, RoutedEventArgs e)
    {
        if (_listScrollViewer is not null) return;
        _listScrollViewer = FindDescendantScrollViewer(nameMessageList);
        if (_listScrollViewer is not null)
            _listScrollViewer.ViewChanged += OnListScrollViewerViewChanged;
    }

    void OnUnloaded(object sender, RoutedEventArgs e)
    {
        if (_listScrollViewer is not null)
        {
            _listScrollViewer.ViewChanged -= OnListScrollViewerViewChanged;
            _listScrollViewer = null;
        }
    }

    void OnListScrollViewerViewChanged(object? sender, ScrollViewerViewChangedEventArgs e)
    {
        if (sender is not ScrollViewer sv) return;
        var scrollable = sv.ScrollableHeight;
        if (double.IsNaN(scrollable) || double.IsInfinity(scrollable)) return;
        var distanceToBottom = scrollable - sv.VerticalOffset;
        _shouldAutoScroll = distanceToBottom <= AutoScrollBottomThreshold;
    }

    public Task ScrollHistoryToBottomAsync(bool animated = true, bool force = false)
    {
        if (!force && !_shouldAutoScroll) return Task.CompletedTask;

        if (DispatcherQueue.HasThreadAccess)
            return RunScrollCoreAsync(animated);

        var tcs = new TaskCompletionSource();
        if (!DispatcherQueue.TryEnqueue(() => _ = RunScrollWithCompletionAsync(tcs, animated)))
            tcs.TrySetException(new InvalidOperationException("DispatcherQueue.TryEnqueue failed."));
        return tcs.Task;
    }

    async Task RunScrollWithCompletionAsync(TaskCompletionSource tcs, bool animated)
    {
        try
        {
            await RunScrollCoreAsync(animated);
            tcs.TrySetResult();
        }
        catch (Exception ex)
        {
            tcs.TrySetException(ex);
        }
    }

    async Task RunScrollCoreAsync(bool animated)
    {
        await Task.Delay(16);
        if (_listScrollViewer is null)
            _listScrollViewer = FindDescendantScrollViewer(nameMessageList);
        if (_listScrollViewer is null) return;
        var y = _listScrollViewer.ScrollableHeight;
        if (double.IsNaN(y) || double.IsInfinity(y)) return;
        _listScrollViewer.ChangeView(null, y, null, disableAnimation: !animated);
    }

    void nameDraftInput_KeyDown(object sender, KeyRoutedEventArgs e)
    {
        if (e.Key != VirtualKey.Enter) return;

        var shift = InputKeyboardSource.GetKeyStateForCurrentThread(VirtualKey.Shift);
        if (shift.HasFlag(CoreVirtualKeyStates.Down))
        {
            ViewModel.Draft += Environment.NewLine;
            nameDraftInput.SelectionStart = ViewModel.Draft.Length;
        }
        else
        {
            ViewModel.SendCommand.Execute(null);
        }

        e.Handled = true;
    }

    static ScrollViewer? FindDescendantScrollViewer(DependencyObject? root)
    {
        if (root is null) return null;
        var count = VisualTreeHelper.GetChildrenCount(root);
        for (var i = 0; i < count; i++)
        {
            var child = VisualTreeHelper.GetChild(root, i);
            if (child is ScrollViewer sv) return sv;
            var nested = FindDescendantScrollViewer(child);
            if (nested is not null) return nested;
        }
        return null;
    }
}

/// <summary>供业务层依赖的滚底抽象，与具体对话来源解耦。</summary>
public interface IConversationScrollTarget
{
    Task ScrollHistoryToBottomAsync(bool animated = true, bool force = false);
}

public sealed class DemoChatMessage
{
    public required string Sender { get; init; }
    public required string Body { get; set; }
}

public partial class GenericConversationDemoViewModel : ObservableObject
{
    readonly IConversationScrollTarget _scrollTarget;
    readonly DispatcherQueue _dispatcher;

    [ObservableProperty] public partial string SessionTitle { get; set; } = "通用对话 Demo（无 AI）";
    [ObservableProperty] public partial string Draft { get; set; } = string.Empty;
    [ObservableProperty] public partial ObservableCollection<DemoChatMessage> Messages { get; set; } = [];

    public GenericConversationDemoViewModel(IConversationScrollTarget scrollTarget, DispatcherQueue dispatcher)
    {
        _scrollTarget = scrollTarget;
        _dispatcher = dispatcher;
    }

    [RelayCommand]
    async Task Send()
    {
        var text = Draft?.TrimEnd();
        if (string.IsNullOrWhiteSpace(text)) return;

        Messages.Add(new DemoChatMessage { Sender = "我", Body = text });
        Draft = string.Empty;
        _ = _scrollTarget.ScrollHistoryToBottomAsync(animated: true, force: true);

        var assistant = new DemoChatMessage { Sender = "对方", Body = string.Empty };
        Messages.Add(assistant);

        await SimulateStreamAppendAsync(assistant, "这是模拟流式输出：逐字追加正文。\n（未上滑时应自动跟随滚底）");
        _ = _scrollTarget.ScrollHistoryToBottomAsync(animated: true, force: false);
    }

    /// <summary>模拟流式：后台等待间隔，每次改 UI 与滚底都回到 UI 线程（与真实 AI 回调接线方式一致）。</summary>
    async Task SimulateStreamAppendAsync(DemoChatMessage target, string fullText)
    {
        foreach (var ch in fullText)
        {
            await Task.Delay(12).ConfigureAwait(false);
            var chLocal = ch;
            var tcs = new TaskCompletionSource();
            if (!_dispatcher.TryEnqueue(() =>
            {
                try
                {
                    target.Body += chLocal;
                    _ = _scrollTarget.ScrollHistoryToBottomAsync(animated: false, force: false);
                }
                finally
                {
                    tcs.TrySetResult();
                }
            }))
                tcs.TrySetException(new InvalidOperationException("DispatcherQueue.TryEnqueue failed."));
            await tcs.Task.ConfigureAwait(false);
        }
    }
}
```

**说明**：

- 真实项目的流式回调若在 **非 UI 线程**：先在后台解析/拼字符串，再用 **`DispatcherQueue.TryEnqueue`**（或等价物）回到 UI 线程 **`ObservableCollection` / 可绑定属性赋值**，并调用 **`ScrollHistoryToBottomAsync(false, false)`**；高频时可再加 **30~60ms 节流**（见 MAUI 知识库同条原则）。

---

## 附录 C：回归测试清单（WinUI）

- 空列表首次输入发送。
- 连续发送多条；`Draft` 清空是否正常。
- 输入很长一行，确认 `Wrap` + 纵向滚动条。
- **Enter** 发送、**Shift+Enter** 多行，再 Enter 发送。
- 流式（或模拟）输出时停在底部：持续跟随。
- 输出过程中上滑：不再强制回底；手动滑回底部后恢复跟随。
- 发送瞬间列表应 **`force` 回底**。
- 页面导航离开再进入（若启用 `NavigationCacheMode`）：状态是否符合预期。

---

## 本次结果（记录）

- 已整理：**WinUI 3 对话页布局、`TextBox` 绑定、`Enter`/`Shift+Enter`、`e.Handled`、ListView 内 `ScrollViewer` 滚底状态机、`DispatcherQueue`**。
- 已提供：**与 AI 解耦的接口 + 完整 XAML/C# Demo 骨架**（命名空间需按项目替换）。
- 与 `MAUI聊天页键盘适配与自动滚动.md` 的关系：**同一套产品语义（有条件滚底 + 用户强制滚底）**；实现面 MAUI 用 `ScrollView.Scrolled`，WinUI 用 **`ScrollViewer.ViewChanged`** 与 **`ChangeView`**。

后续若你方 `ChatPage` 再演进（例如光标处插换行、流式节流），建议 **同步更新本文「设计取舍」与 Demo** 一节，保持知识库与仓库行为一致。
