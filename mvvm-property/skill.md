---
name: mvvm-property
description: Generates C# notification properties using Prism BindableBase or CommunityToolkit.Mvvm ObservableObject/attribute patterns
allowed-tools: [Read, Write, Edit, Grep, Glob, PowerShell]
---

# MVVM 通知属性生成器

这个 skill 帮助你在 WPF/Avalonia/MAUI 项目中生成带 INotifyPropertyChanged 通知的属性，支持 **CommunityToolkit.Mvvm** (推荐) 和 **Prism.Mvvm** 两种框架。

## 触发方式

输入 `/mvvm-property` 或描述需要生成的属性。

## 使用流程

### 1. 识别框架

首先检查项目中已引用的 NuGet 包或 ViewModel 基类：
- 搜索 `CommunityToolkit.Mvvm` 包引用
- 搜索 `Prism.Mvvm` / `Prism.Core` 包引用
- 查看现有 ViewModel 继承自 `ObservableObject` 还是 `BindableBase`

如果找不到或无法判断，询问用户使用哪种框架。

### 2. 确认需求

向用户确认以下信息：
- **框架选择**：CommunityToolkit.Mvvm 或 Prism
- **属性类型、名称**
- **是否需要触发其他属性的通知** (`[NotifyPropertyChangedFor]`)
- **是否需要自定义 setter 逻辑**（验证、联动等）
- **是否要在特定文件的特定位置插入**

### 3. 生成代码

#### 模式 A：CommunityToolkit.Mvvm (推荐)

CommunityToolkit.Mvvm 有四种 `[ObservableProperty]` 标注方式，根据场景选择：

##### A1. 字段标注（最经典、兼容性最好）

`[ObservableProperty]` 加在 private 字段上，source generator 自动生成同名属性（去下划线、PascalCase）。

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class MyViewModel : ObservableObject
{
    [ObservableProperty]
    private string _userName;

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(FullName))]
    private string _firstName;

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(FullName))]
    private string _lastName;

    public string FullName => $"{FirstName} {LastName}";
}
```

##### A2. 分部属性标注（简洁、无需字段）

`[ObservableProperty]` 加在 **partial** auto-property 上，生成器自动注入通知逻辑。
基于 C# 13 的 **分部属性 (partial properties)** 功能。

```csharp
public partial class MyViewModel : ObservableObject
{
    [ObservableProperty]
    public partial string Title { get; set; }   // 注意 property 也要 partial
}
// 等价于手写 SetProperty，但不需声明字段
```

> **版本要求：** CommunityToolkit.Mvvm ≥ v8.4.0，.NET 9 SDK / C# 13+
> - v8.4.0：需要 `<LangVersion>preview</LangVersion>`
> - v8.4.1+：`<LangVersion>14.0</LangVersion>` 或 `latest`

##### A3. 半自动属性 + `field` 关键字（需要自定义 get/set 时）

同时使用 C# 13 的 `field` 关键字（semi-auto properties）和 `[ObservableProperty]`，
生成器自动提供 backing field，同时你可以自定义 getter/setter：

```csharp
public partial class MyViewModel : ObservableObject
{
    [ObservableProperty]
    public string Title
    {
        get => field;
        set => SetProperty(ref field, value);
    }
}
```

> **版本要求：** 与 A2 相同（CommunityToolkit.Mvvm ≥ v8.4.0, .NET 9 / C# 13+），
> 额外要求 C# 13 的 `field` 关键字功能。
> 实际上 A2/A3 底层依赖同一个 partial property 机制，A3 只是手动展开了 get/set。

##### A4. Hook 方法（值变更时执行额外逻辑）

仍然用字段标注 `[ObservableProperty]`，通过 `partial void OnXxxChanged` 挂钩：

```csharp
[ObservableProperty]
private string _name;

partial void OnNameChanged(string oldValue, string newValue)
{
    // 验证、联动等自定义逻辑
}
```

只有当 hook 无法满足需求时（如需要拦截赋值、条件性阻止变更），才退化到 A3 的半自动属性或完全手写 `SetProperty`。

#### 模式 B：Prism.Mvvm

Prism 始终使用 `SetProperty` 方法（继承自 `BindableBase`）：

```csharp
using Prism.Mvvm;

public class MyViewModel : BindableBase
{
    private string _userName;
    public string UserName
    {
        get => _userName;
        set => SetProperty(ref _userName, value);
    }

    private string _firstName;
    public string FirstName
    {
        get => _firstName;
        set
        {
            if (SetProperty(ref _firstName, value))
            {
                RaisePropertyChanged(nameof(FullName));
            }
        }
    }

    private string _lastName;
    public string LastName
    {
        get => _lastName;
        set
        {
            if (SetProperty(ref _lastName, value))
            {
                RaisePropertyChanged(nameof(FullName));
            }
        }
    }

    public string FullName => $"{FirstName} {LastName}";
}
```

### 4. 插入代码

使用 Edit 工具精确插入到目标 ViewModel 文件中。遵循现有代码风格（缩进、空行等）。

## 关键约定

- **类必须是 `partial`** 才能使用 CommunityToolkit.Mvvm 的 `[ObservableProperty]`
- **字段标注 (A1)**：小写驼峰加下划线前缀 `_fieldName`，生成器转为 PascalCase 属性名。适用于所有版本，无特殊要求
- **属性标注 (A2/A3)**：需要 **partial property**（C# 13+），所以在属性上也必须加 `partial` 关键字。要求 CommunityToolkit.Mvvm ≥ v8.4.0 + .NET 9 SDK
- **`field` 关键字 (A3)**：额外需要 C# 13 的 semi-auto property 功能
- **验证**：需要验证时使用 `ObservableValidator` 基类
- **依赖属性通知**：推荐 `[NotifyPropertyChangedFor]` 而非手动调用 `OnPropertyChanged`
- 检查生成的代码是否符合项目现有模式

## NuGet 包名参考

| 框架 | 包名 |
|------|------|
| CommunityToolkit.Mvvm | `CommunityToolkit.Mvvm` |
| Prism.Core | `Prism.Core` |
| Prism.Mvvm (WPF) | `Prism.Wpf` |
| Prism.Mvvm (Uno) | `Prism.Uno` |
| Prism.Mvvm (MAUI) | `Prism.Maui` |
