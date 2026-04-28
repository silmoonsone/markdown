# MAUI Chat 键盘适配与自动滚动通用知识库（iOS）

## 文档用途
- 这是 MAUI 聊天页在 iOS 上“键盘适配 + 流式自动滚动”的通用知识库。
- 目标是让任何项目接手者不看历史聊天，只看此文档即可落地同类能力。

## 术语说明
- 键盘“避让”：键盘弹出时，页面自动让出空间，避免输入区或内容被遮挡。
- 自动滚动“跟随”：列表在底部时随新消息持续滚动；用户手动上滑后暂停跟随；回到底部后恢复跟随。

## 通用需求背景
- 页面是聊天结构：上方聊天记录、下方输入栏。
- iOS 真机键盘弹出时：
  - 输入栏不能被遮挡；
  - 输入栏不能抬升过高；
  - 聊天记录区域要随键盘压缩，而不是和输入栏视觉重叠。
- 需要保留点击空白收起键盘：`HideSoftInputOnTapped="True"`。
- AI 流式输出时聊天列表要自动滚到底部；但用户手动上滑查看历史时，自动滚动应暂停；用户回到底部后自动滚动恢复。

## 最终稳定实现（当前基线）

### 1) 页面布局采用 `Grid(*, Auto)`
- 第 0 行：历史内容滚动区（`ScrollView`）。
- 第 1 行：输入栏（`InputBar`）。

### 2) iOS 键盘事件 + 布局底部内边距方案
- 页面进入时（iOS）关闭 MAUI 自动键盘滚动：`KeyboardAutoManagerScroll.Disconnect()`。
- 页面离开时恢复：`KeyboardAutoManagerScroll.Connect()`。
- 监听：
  - `UIKeyboard.Notifications.ObserveWillChangeFrame`
  - `UIKeyboard.Notifications.ObserveWillHide`
- 键盘变化时不平移输入栏，而是调整 `RootLayout.Padding.Bottom`：
  - 根据 `InputBar` 和键盘的实际重叠量计算 `requiredInset`。
  - 这样会触发整体重排，历史区被压缩，避免“内容和输入框叠一起”。

### 3) 聊天历史自动滚动（含“上滑暂停，回底恢复”）
- 在页面维护状态：
  - `const double AutoScrollBottomThreshold = 24;`
  - `bool shouldAutoScroll = true;`
- `ScrollView` 绑定 `Scrolled` 事件，实时计算“距离底部”。
  - `distanceToBottom <= threshold` 时，`shouldAutoScroll = true`。
  - 用户上滑后距离底部大于阈值时，`shouldAutoScroll = false`。
- 封装 `ScrollHistoryToBottomAsync(bool animated = true, bool force = false)`：
  - 正常流式输出时调用：`force = false`，遵守 `shouldAutoScroll`。
  - 用户主动发送消息时调用：`force = true`，强制滚到底部。
- 流式输出每个 chunk 更新后触发滚动：
  - 输出中：`_ = page.ScrollHistoryToBottomAsync(false);`
  - 输出完成：`_ = page.ScrollHistoryToBottomAsync();`

## 关键代码（可复用模板）

### A. `Chat.xaml`（模板结构）
```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:ios="clr-namespace:Microsoft.Maui.Controls.PlatformConfiguration.iOSSpecific;assembly=Microsoft.Maui.Controls"
             x:Class="Silmoon.Intelligence.MauiClient.Pages.Chat"
             xmlns:page="clr-namespace:Silmoon.Intelligence.MauiClient.Pages"
             xmlns:models="clr-namespace:Silmoon.Intelligence.MauiClient.Models"
             x:DataType="page:ChatViewModel"
             HideSoftInputOnTapped="True"
             Title="Chat">
    <Grid x:Name="RootLayout" RowDefinitions="*,Auto" RowSpacing="0">
        <ScrollView x:Name="HistoryScrollView" Grid.Row="0" VerticalScrollBarVisibility="Always" Padding="8" Scrolled="HistoryScrollView_Scrolled">
            <StackLayout BindableLayout.ItemsSource="{x:Binding Items}">
                <BindableLayout.ItemTemplate>
                    <DataTemplate x:DataType="models:ChatItem">
                        <Border Padding="5" Margin="0,0,0,10" BackgroundColor="{AppThemeBinding Dark={StaticResource Gray950}, Light={StaticResource Gray100}}" StrokeThickness="0" StrokeShape="RoundRectangle 12">
                            <StackLayout>
                                <Label Text="{x:Binding Role}" FontAttributes="Bold" />
                                <Label Text="{x:Binding Content}" LineBreakMode="WordWrap" />
                            </StackLayout>
                        </Border>
                    </DataTemplate>
                </BindableLayout.ItemTemplate>
                <Label Text="{x:Binding Output}" LineBreakMode="WordWrap" />
            </StackLayout>
        </ScrollView>

        <Grid x:Name="InputBar" Grid.Row="1" ColumnDefinitions="*,Auto" ColumnSpacing="8" Padding="8">
            <Entry Grid.Column="0" Text="{x:Binding Input}" HorizontalOptions="Fill" />
            <Button Grid.Column="1" Text="发送" Command="{x:Binding ChatCommand}" />
        </Grid>
    </Grid>
</ContentPage>
```

### B. `Chat.xaml.cs`（键盘适配 + 自动滚动关键段）
```csharp
#if IOS
using Foundation;
using Microsoft.Maui.Platform;
using UIKit;
#endif

public partial class Chat : ContentPage
{
    const double AutoScrollBottomThreshold = 24;
    bool shouldAutoScroll = true;

#if IOS
    NSObject? keyboardWillChangeFrameObserver;
    NSObject? keyboardWillHideObserver;
#endif

    public Task ScrollHistoryToBottomAsync(bool animated = true, bool force = false)
    {
        if (!force && !shouldAutoScroll)
            return Task.CompletedTask;

        return MainThread.InvokeOnMainThreadAsync(async () =>
        {
            // 等待本帧布局更新，避免内容尚未测量时滚动失败
            await Task.Delay(16);
            await HistoryScrollView.ScrollToAsync(0, double.MaxValue, animated);
        });
    }

    void HistoryScrollView_Scrolled(object? sender, ScrolledEventArgs e)
    {
        var maxScrollY = Math.Max(0, HistoryScrollView.ContentSize.Height - HistoryScrollView.Height);
        var distanceToBottom = maxScrollY - e.ScrollY;
        shouldAutoScroll = distanceToBottom <= AutoScrollBottomThreshold;
    }

    protected override void OnAppearing()
    {
        base.OnAppearing();
#if IOS
        KeyboardAutoManagerScroll.Disconnect();
        RegisterKeyboardObservers();
#endif
    }

    protected override void OnDisappearing()
    {
#if IOS
        UnregisterKeyboardObservers();
        KeyboardAutoManagerScroll.Connect();
        RootLayout.Padding = new Thickness(0);
#endif
        base.OnDisappearing();
    }

#if IOS
    void RegisterKeyboardObservers()
    {
        if (keyboardWillChangeFrameObserver is not null) return;

        keyboardWillChangeFrameObserver =
            UIKeyboard.Notifications.ObserveWillChangeFrame((_, args) => OnKeyboardWillChangeFrame(args));
        keyboardWillHideObserver =
            UIKeyboard.Notifications.ObserveWillHide((_, args) => OnKeyboardWillHide(args));
    }

    void UnregisterKeyboardObservers()
    {
        keyboardWillChangeFrameObserver?.Dispose();
        keyboardWillChangeFrameObserver = null;
        keyboardWillHideObserver?.Dispose();
        keyboardWillHideObserver = null;
    }

    void OnKeyboardWillChangeFrame(UIKeyboardEventArgs args)
    {
        MainThread.BeginInvokeOnMainThread(async () =>
        {
            if (Handler?.PlatformView is not UIView pageView || pageView.Window is null)
                return;
            if (InputBar.Handler?.PlatformView is not UIView inputBarView)
                return;

            // 同一坐标系下计算重叠量
            var keyboardFrameInPage = pageView.ConvertRectFromView(args.FrameEnd, null);
            var inputBarFrameInPage = pageView.ConvertRectFromView(inputBarView.Bounds, inputBarView);
            var overlap = Math.Max(0, inputBarFrameInPage.Bottom - keyboardFrameInPage.Top);

            // 用布局内边距触发重排（历史区会被压缩）
            var requiredInset = Math.Max(0, overlap + 2);
            RootLayout.Padding = new Thickness(0, 0, 0, requiredInset);
            await Task.CompletedTask;
        });
    }

    void OnKeyboardWillHide(UIKeyboardEventArgs args)
    {
        MainThread.BeginInvokeOnMainThread(async () =>
        {
            RootLayout.Padding = new Thickness(0);
            await Task.CompletedTask;
        });
    }
#endif
}
```

### C. `ChatViewModel`（流式输出触发自动滚动）
```csharp
[ObservableProperty]
public partial ObservableCollection<ChatItem> Items { get; set; } = [];

private Task NativeChatClient_OnStreamOutputCompleted(Result result)
{
    var lastChatItem = Items.LastOrDefault();
    lastChatItem.Content += $"\r\n[finish {result.FinishReason}]";
    _ = page.ScrollHistoryToBottomAsync();
    return Task.CompletedTask;
}

private void NativeChatClient_OnStreamOutput(StateSet<bool, Chunk> chunkState)
{
    if (chunkState.State)
    {
        // ...持续拼接 lastChatItem.Content
        _ = page.ScrollHistoryToBottomAsync(false);
    }
}

[RelayCommand]
public async Task Chat()
{
    Items.Add(new ChatItem { Role = Role.User, Content = Input });
    Items.Add(new ChatItem { Role = Role.Assistant, Content = string.Empty });
    _ = page.ScrollHistoryToBottomAsync(force: true);
    // ...发送请求
}
```

### D. `App.xaml.cs`（与键盘相关的通用注意点）
```csharp
protected override Window CreateWindow(IActivationState? activationState)
{
    var window = new Window(new AppShell());

    // 仅桌面端固定窗口尺寸，移动端不要固定
    if (DeviceInfo.Platform == DevicePlatform.WinUI || DeviceInfo.Platform == DevicePlatform.MacCatalyst)
    {
        window.Width = 1200;
        window.Height = 800;
    }
    return window;
}
```

## 关键原则（必须遵守）
- 不要同时启用“系统自动键盘滚动 + 自定义位移”，否则容易叠加导致抬升异常。
- 避让计算必须在同一坐标系中完成（通过 `ConvertRectFromView`）。
- 聊天页更推荐“调整布局内边距”而非“直接平移输入栏”，因为前者会压缩历史区，视觉更自然。
- 自动滚动必须是“有条件自动”：用户上滑时暂停，回到底部才恢复，避免打断阅读。

## 一次性落地补充（建议纳入实现）

### 1) 生命周期与事件订阅防重复
- `NativeChatClient` 相关事件建议在 `Loaded` 订阅、`Unloaded` 解除订阅，避免页面反复进入后重复回调。
- 当前代码里 `Loaded/Unloaded` 解除逻辑有注释痕迹，后续若重构需确认“订阅与解除是一一对应”的。
- 经验法则：页面类负责 UI 生命周期，ViewModel 只做纯状态更新更稳。

### 2) 流式输出空引用保护
- `Items.LastOrDefault()` 在极端时序下可能为 `null`（如异常中断、列表被清空）。
- 对 `lastChatItem` 写入前建议判空并快速返回，避免偶发崩溃。
- 发送逻辑建议确保“先创建 assistant item，再发请求”，当前已符合。

### 3) 滚动触发节流
- 流式 chunk 很密集时，`ScrollToAsync` 调用频率会很高。
- 建议增加轻量节流（如 30~60ms 最小间隔）减少滚动抖动与主线程压力。
- 当前 `Task.Delay(16)` 是有效的基础优化，可在高频模型输出时再升级节流。

### 4) 用户意图优先策略
- 自动滚动只在 `shouldAutoScroll=true` 时执行（当前已实现）。
- `force=true` 仅用于“用户主动发送后跳到底部”（当前已实现），不要用于流式过程。
- 阈值 `AutoScrollBottomThreshold=24` 可按设备 DPI 微调（常用区间 `16~48`）。

### 5) 输入区交互体验细节
- 发送按钮禁用条件：空输入时禁用（当前已通过 `IsNotNullOrEmptyConvert` 使用）。
- 可补充“按回车发送”（可选）并保留“空白点击收键盘”（当前已实现 `HideSoftInputOnTapped="True"`）。
- 若增加多行输入框，键盘避让公式不变，只需注意输入栏高度变化对 `overlap` 的影响。

### 6) 长会话性能建议
- `BindableLayout + StackLayout` 在超长消息记录时会有测量压力；后续可迁移 `CollectionView` 做虚拟化。
- 对超长单条消息可考虑分段渲染或懒加载，降低重排成本。
- 若有 Markdown 富文本渲染，优先异步预处理后再上主线程赋值。

## 项目特有代码替换表（落地前先做映射）
- `IntelligenceService`：替换为你自己的聊天服务接口（如 `IChatService`）。
- `NativeChatClient.OnStreamOutput*`：替换为你自己的流式回调事件或 `IAsyncEnumerable`。
- `ChatItem` 字段（`Role/Content`）：按你的消息模型字段改名（如 `Sender/Text`）。
- `IsNotNullOrEmptyConvert`：若无该转换器，用 `string.IsNullOrWhiteSpace` 对应的命令可执行状态替代。
- `DisplayAlertAsync` / `Input` 校验：按项目 UI 框架与交互规范替换。
- 颜色与样式资源（`Gray950/Gray100`）：按你的主题资源表替换。

## 最小可移植清单（跨项目复制时）
1. 复制 `Grid(*,Auto)` + `InputBar` + `HistoryScrollView` 布局骨架。
2. 复制 iOS 键盘 frame 监听与 `RootLayout.Padding.Bottom` 计算逻辑。
3. 复制 `shouldAutoScroll` 状态机（`Scrolled` 事件 + 阈值判断 + `force`）。
4. 将流式输出回调映射到“追加消息 + 条件滚动”调用点。
5. 把项目特有服务/模型/转换器按“替换表”换成目标项目实现。

## 常见问题与处理
- 输入栏被键盘遮挡：检查是否正确注册 `ObserveWillChangeFrame`。
- 输入栏抬得过高：检查是否叠加了系统自动滚动；减小 `overlap + 2` 中缓冲值。
- 切换输入法后才正常：通常是键盘 frame 处理时机不对，优先用 `WillChangeFrame`。
- 页面离开后下次布局异常：确认 `OnDisappearing` 里恢复了 `KeyboardAutoManagerScroll.Connect()` 且 `RootLayout.Padding` 清零。
- 流式输出不自动滚到底：确认输出更新后有调用 `ScrollHistoryToBottomAsync(false)`。
- 用户上滑时仍被强制拉回底部：确认 `force` 只在主动发送消息时使用。
- 长时间聊天后滚动卡顿：优先评估 `CollectionView` 虚拟化替代 `BindableLayout`。
- 偶发空引用：重点检查 `lastChatItem` 是否为空、事件是否重复订阅导致竞态。

## 回归测试清单
- 首次进入页面直接点输入框。
- 连续收起/弹出键盘 5 次。
- 切换输入法前后对比。
- 历史记录为空、少量、大量三种状态。
- 有无发送按钮点击后焦点变化场景。
- AI 连续流式输出时，保持在底部会持续跟随最新内容。
- AI 连续流式输出时，手动上滑后不会被强制拉回底部。
- 流式输出中手动滚到底部后，会恢复自动跟随。

## 本次结果（记录）
- 已实现：输入栏与内容区域分离，聊天记录会随键盘压缩。
- 已保留：`HideSoftInputOnTapped="True"`。
- 已实现：流式输出自动滚动 + 用户上滑暂停 + 回到底部恢复。
- iOS 目标已编译通过：`dotnet build -f net10.0-ios`。

## 实施优先级（下次从零开始可按此顺序）
1. 先搭建 `Grid(*,Auto)` 聊天布局与 `InputBar` 命名。
2. 接入 iOS 键盘 frame 监听与 `RootLayout.Padding.Bottom` 方案。
3. 接入 `Scrolled` + `shouldAutoScroll` 状态机与 `force` 策略。
4. 接入流式输出写入与滚动调用点（输出中、输出完成、发送后）。
5. 做生命周期/空引用/节流加固，再跑回归清单。
