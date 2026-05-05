# C# 编码习惯（给大模型）

本文档用于约束大模型在我的 C# 代码库中补全、修改、重构时的行为。  
目标是：保持我一贯的编码风格与信息密度，不做无关改动。

## 1. 总体原则

- 信息密度优先：尽量让一个屏幕看到更多有效信息。
- 在可读性可接受的前提下，优先单行写法，不轻易拆成多行。
- 是否“太长需要换行”偏主观判断，只有明显过长时才换行。
- 不做与任务无关的风格化改动。

## 2. if / else 风格

- 单语句分支可不加大括号。
- 只要分支体达到两行及以上，必须使用 `{}`。
- 默认 `if` 和 `else` 分行写；只有在两边都非常短时，才可同一行。
- 早返回风格是常态，例如：
  - `if (cond) return null;`
  - `if (cond) return;`
- 单语句但过长时允许换行，例如：

```csharp
if (condition)
    DoSomethingVeryLongNameOrExpression();
else return null;
```

## 3. 参数与列表换行偏好

- 参数较多时，定义与调用都优先保持单行。
- 该偏好不仅限于方法，也适用于类似“参数/元素列表”场景（如构造调用、特性参数等）。
- 只有在明显过长、影响阅读时才换行，不按机械阈值强制拆行。

## 4. 链式调用与 LINQ 换行偏好

- 链式调用默认尽量保持单行，不要为了“看起来整齐”主动拆行。
- 尤其是可读性仍然清晰时，`Split(...).Select(...).Where(...)` 这类链式表达式应保持一行。
- 只有在表达式明显过长、已经影响阅读时才允许换行。
- 明确偏好示例：
  - 不偏好：
    ```csharp
    var encodedPath = string.Join('/', filePath
        .Split('/', StringSplitOptions.RemoveEmptyEntries)
        .Select(Uri.EscapeDataString));
    ```
  - 偏好：
    ```csharp
    var encodedPath = string.Join('/', filePath.Split('/', StringSplitOptions.RemoveEmptyEntries).Select(Uri.EscapeDataString));
    ```

## 5. 复用 Silmoon 库（长期习惯）

- 我长期使用自有可复用库/包（`Silmoon`），这是稳定习惯。
- 判空优先使用扩展方法风格：
  - 字符串：`"".IsNullOrEmpty()`
  - 集合/数组等类型也会使用同名 `IsNullOrEmpty` 扩展方法
- 模型在阅读/生成代码时，应优先沿用我的库能力，不要在每个项目重复造轮子。

参考签名（用于让模型理解语义，不依赖固定路径）：

- `public static bool IsNullOrEmpty(this string value)`
- `public static bool IsNullOrEmpty<T>(this ICollection<T> array)`（以及同类集合扩展）
- `public class StateSet<TState> { public TState State { get; set; } public string Message { get; set; } }`
- `public class StateSet<TState, TData> : StateSet<TState> { public TData Data { get; set; } }`

## 6. StateSet 返回值语义

- 当代码返回 `StateSet` 相关类型时，请按我的统一语义理解：
  - `State`：状态
  - `Message`：说明信息
  - `Data`（泛型版本）：附带数据
- 不要把它当作一次性临时结构；它是我的通用返回模型。

## 7. 注释规则（硬约束）

- 默认不要主动添加注释。
- 除非代码确实难以理解，否则不写注释。
- 未修改区域严禁“顺手补注释”。
- 仅允许在本次新增/修改的代码片段中，按需添加少量注释。
- 注释应解释复杂意图，不解释显而易见的代码。

## 8. 对大模型的执行要求

- 先遵循现有风格，再考虑个人偏好的“最佳实践”。
- 不要因“看起来更规范”而大规模改写无关代码。
- 输出代码时优先保持与周边代码一致的格式与调用习惯（尤其是 `Silmoon` 扩展与 `StateSet` 语义）。

---

后续可持续追加新习惯；若与旧规则冲突，以最新补充为准。
