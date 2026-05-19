---
name: csharp-config
description: C# configuration patterns covering options pattern, IConfiguration, IOptionsMonitor, IOptionsSnapshot, IConfigurationSource, NuGet packages, and usage in non-ASP.NET projects (console, WPF, WinForms)
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash]
---

# C# 配置模式与选项模式

这个 skill 涵盖现代 .NET 中配置管理的完整知识：选项模式、IConfiguration 系统、IOptionsMonitor 热更新、自定义配置源、非 ASP.NET 项目集成，以及 WPF 中的特殊用法。

## 概述

现代 .NET 推荐使用 **选项模式 (Options Pattern)** + **强类型配置类**，通过 `IConfiguration` 绑定 JSON/环境变量，再通过依赖注入提供给服务。

## 1. 配置类的定义与注册

### 定义配置类

```csharp
public class DatabaseOptions
{
    public const string SectionName = "Database"; // 匹配 JSON 节名
    public string ConnectionString { get; set; } = string.Empty;
    public int MaxRetryCount { get; set; }
    public bool EnableLogging { get; set; }
}
```

### appsettings.json

```json
{
  "Database": {
    "ConnectionString": "Server=localhost;Database=MyDb",
    "MaxRetryCount": 3,
    "EnableLogging": true
  }
}
```

### 注册方式

```csharp
// 方式1：绑定到 IConfiguration 节
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));

// 方式2：手动绑定并注册为单例
var dbOptions = builder.Configuration.GetSection("Database").Get<DatabaseOptions>();
builder.Services.AddSingleton(dbOptions);

// 方式3：带验证的注册
builder.Services.AddOptions<DatabaseOptions>()
    .Bind(builder.Configuration.GetSection("Database"))
    .ValidateDataAnnotations()
    .Validate(options => {
        return options.MaxRetryCount != 5 || options.EnableLogging;
    }, "当重试次数为5时必须启用日志");
```

### 配置验证

```csharp
public class DatabaseOptions
{
    [Required(ErrorMessage = "连接字符串不能为空")]
    public string ConnectionString { get; set; }

    [Range(1, 10, ErrorMessage = "重试次数必须在1-10之间")]
    public int MaxRetryCount { get; set; }

    [RegularExpression(@"^[A-Z]+$")]
    public string DatabaseType { get; set; }
}
```

### 嵌套配置类

```csharp
public class StorageOptions
{
    public BlobConfig Blob { get; set; }
}
public class BlobConfig
{
    public string ContainerName { get; set; }
}
// JSON 路径: "Storage:Blob:ContainerName"
```

---

## 2. 三种 Options 访问接口

| 接口 | 生命周期 | 配置变化响应 | 推荐场景 |
|------|---------|-------------|---------|
| `IOptions<T>` | 单例 | 不响应，始终是初始值 | 静态配置、启动后不改变的配置 |
| `IOptionsSnapshot<T>` | 作用域/瞬态 | 每次请求获取最新值 | Web 应用中需要热更新的配置 |
| `IOptionsMonitor<T>` | 单例 | 主动监听变化并通知 | 需要实时响应配置变更的场景 |

### 使用对比

```csharp
public class TestController : ControllerBase
{
    private readonly IOptions<MyConfig> _options;
    private readonly IOptionsSnapshot<MyConfig> _snapshot;
    private readonly IOptionsMonitor<MyConfig> _monitor;

    public TestController(
        IOptions<MyConfig> options,
        IOptionsSnapshot<MyConfig> snapshot,
        IOptionsMonitor<MyConfig> monitor)
    {
        _options = options;
        _snapshot = snapshot;
        _monitor = monitor;
    }

    [HttpGet]
    public IActionResult Test()
    {
        var a = _options.Value.Timeout;      // 不会变（启动时快照）
        var b = _snapshot.Value.Timeout;     // 下次请求会变
        var c = _monitor.CurrentValue.Timeout; // 实时变化
        return Ok(new { a, b, c });
    }
}
```

---

## 3. IOptionsMonitor 详解

### 热更新原理

`IOptionsMonitor` 监听 `IConfiguration` 系统发出的变更通知（`IChangeToken`）。当配置文件变化时，配置系统检测到文件变化并触发令牌，`IOptionsMonitor` 订阅此通知。**关键前提**：注册配置源时设置 `reloadOnChange: true`。

### 使用 CurrentValue 获取最新值

```csharp
public class MySingletonService
{
    private readonly AppSettings _currentSettings;

    public MySingletonService(IOptionsMonitor<AppSettings> optionsMonitor)
    {
        _currentSettings = optionsMonitor.CurrentValue; // 总是最新值
    }
}
```

### 订阅配置变更回调

```csharp
public class MyService : IDisposable
{
    private readonly IDisposable? _changeListener;

    public MyService(IOptionsMonitor<AppSettings> optionsMonitor)
    {
        _changeListener = optionsMonitor.OnChange(updatedSettings =>
        {
            Console.WriteLine($"配置变化！新值: {updatedSettings.NewValue}");
            // 刷新缓存、重新初始化组件等
        });
    }

    public void Dispose()
    {
        _changeListener?.Dispose(); // 记得释放，避免内存泄漏
    }
}
```

### 程序内修改配置并持久化

`IOptionsMonitor` **只读**，不具备修改和保存功能。如需程序内修改配置：

1. **写回文件**：直接修改 `appsettings.json`（`File.WriteAllText`），文件变更会触发 `reloadOnChange`
2. **数据库存储**：将动态配置存于数据库，自定义 `IConfigurationSource` 响应数据库变更

---

## 4. IConfigurationSource 自定义配置源

`IConfigurationSource` 是 .NET 官方接口（`Microsoft.Extensions.Configuration` 命名空间），作为配置源的工厂，负责创建 `IConfigurationProvider`。

### 架构层次

```
IConfigurationBuilder
  └── IConfigurationSource 集合 (Json, Xml, EnvVar, CommandLine...)
      └── Build() → IConfigurationRoot
          └── IConfigurationProvider 集合 (实际读取数据)
```

### 自定义数据库配置源示例

```csharp
// 1. 自定义 Source
public class DatabaseConfigurationSource : IConfigurationSource
{
    public string ConnectionString { get; set; }
    public string Query { get; set; }
    public int ReloadDelay { get; set; } = 30000;

    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new DatabaseConfigurationProvider(this);
    }
}

// 2. 自定义 Provider
public class DatabaseConfigurationProvider : ConfigurationProvider
{
    private readonly DatabaseConfigurationSource _source;
    private Timer? _timer;

    public DatabaseConfigurationProvider(DatabaseConfigurationSource source)
    {
        _source = source;
        if (_source.ReloadDelay > 0)
            _timer = new Timer(Reload, null, _source.ReloadDelay, _source.ReloadDelay);
    }

    private void Reload(object? state)
    {
        Load();
        OnReload(); // 触发变更通知 → IOptionsMonitor 可感知
    }

    public override void Load()
    {
        // 从数据库读取配置键值对
        Data = new Dictionary<string, string?> { ... };
    }
}

// 3. 扩展方法
public static class DatabaseConfigurationExtensions
{
    public static IConfigurationBuilder AddDatabase(
        this IConfigurationBuilder builder,
        string connectionString, string query, int reloadDelay = 30000)
    {
        return builder.Add(new DatabaseConfigurationSource
        {
            ConnectionString = connectionString,
            Query = query,
            ReloadDelay = reloadDelay
        });
    }
}
```

---

## 5. 非 ASP.NET 项目的 NuGet 包

| 类别 | 包名 | 作用 |
|------|------|------|
| 核心基础 | `Microsoft.Extensions.Configuration` | 必备。`IConfiguration`、`ConfigurationBuilder` |
| 配置提供程序 | `Microsoft.Extensions.Configuration.Json` | JSON 文件读取，支持 `reloadOnChange` |
| | `Microsoft.Extensions.Configuration.EnvironmentVariables` | 环境变量 |
| | `Microsoft.Extensions.Configuration.CommandLine` | 命令行参数 |
| | `Microsoft.Extensions.Configuration.Ini` / `.Xml` | INI/XML 格式 |
| 绑定与选项 | `Microsoft.Extensions.Configuration.Binder` | 必备。配置绑定到强类型对象 |
| | `Microsoft.Extensions.Options` | 选项模式核心：`IOptions<>` 等接口 |
| DI 与托管 | `Microsoft.Extensions.Hosting` | **推荐**。一站式集成配置、DI、日志 |
| | `Microsoft.Extensions.DependencyInjection` | 独立 DI 容器 |

---

## 6. 三种集成方式

### 方式 A：轻量级 — 直接构建 ConfigurationBuilder

适合极简工具，不含 DI。

```csharp
var config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
    .AddEnvironmentVariables()
    .Build();

var title = config["AppSettings:Title"];
var timeout = config.GetValue<int>("AppSettings:Timeout", 30);
```

### 方式 B：ServiceCollection + ConfigurationBuilder

中等场景，手动组合 DI 容器和配置系统。

```csharp
IConfiguration config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddEnvironmentVariables()
    .Build();

IServiceCollection services = new ServiceCollection();
services.Configure<MySettings>(config.GetSection("MySettings"));
services.AddSingleton<IConfiguration>(config);
services.AddTransient<IMyService, MyService>();

IServiceProvider serviceProvider = services.BuildServiceProvider();
var myService = serviceProvider.GetRequiredService<IMyService>();
```

### 方式 C：通用主机（推荐）

`Host.CreateDefaultBuilder` 自动处理配置、DI、日志、生命周期。

```csharp
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        services.Configure<MySettings>(context.Configuration.GetSection("MySettings"));
        services.AddSingleton<IMyService, MyService>();
    })
    .Build();

var myService = host.Services.GetRequiredService<IMyService>();
```

`Host.CreateDefaultBuilder` 默认按优先级加载：`appsettings.json` → `appsettings.{Environment}.json` → 用户机密(Dev) → 环境变量 → 命令行参数。

---

## 7. WPF 中使用 IOptionsSnapshot

### 核心挑战

`IOptionsSnapshot<T>` 是作用域服务，而 WPF 的 ViewModel 通常注册为单例。单例服务锁定初始快照，无法热更新。

### 解决方案：手动创建 IServiceScope

```csharp
public class MainViewModel
{
    private readonly IServiceProvider _serviceProvider;

    public MainViewModel(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void ShowSettings()
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            var settings = scope.ServiceProvider
                .GetRequiredService<IOptionsSnapshot<MySettings>>();
            // settings.Value 是最新配置
        }
    }
}
```

### 封装 SettingsService（推荐）

```csharp
public interface ISettingsService
{
    T GetSettings<T>() where T : class;
}

public class SettingsService : ISettingsService
{
    private readonly IServiceProvider _serviceProvider;
    public SettingsService(IServiceProvider sp) => _serviceProvider = sp;

    public T GetSettings<T>() where T : class
    {
        using var scope = _serviceProvider.CreateScope();
        return scope.ServiceProvider.GetRequiredService<IOptionsSnapshot<T>>().Value;
    }
}
```

### WPF App.xaml.cs 启动配置

```csharp
public partial class App : Application
{
    public IServiceProvider ServiceProvider { get; private set; }

    protected override void OnStartup(StartupEventArgs e)
    {
        var configuration = new ConfigurationBuilder()
            .SetBasePath(AppContext.BaseDirectory)
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            .Build();

        var services = new ServiceCollection();
        services.AddSingleton<IConfiguration>(configuration);
        services.Configure<MySettings>(configuration.GetSection("MySettings"));
        services.AddSingleton<ISettingsService, SettingsService>();
        services.AddTransient<MainViewModel>();

        ServiceProvider = services.BuildServiceProvider();

        var mainWindow = new MainWindow
        {
            DataContext = ServiceProvider.GetRequiredService<MainViewModel>()
        };
        mainWindow.Show();
        base.OnStartup(e);
    }
}
```

**注意**：`appsettings.json` 需设置"生成操作"为"内容"，"复制到输出目录"为"如果较新则复制"。

---

## 8. 最佳实践速查

- **类型安全**：用强类型配置类替代松散的字符串键
- **默认值**：在配置类中设置合理的属性默认值
- **不可变配置**：配置类保持 POCO，避免复杂业务逻辑
- **热更新**：需要感知配置变化时用 `IOptionsMonitor<T>`
- **验证**：用 DataAnnotations + `ValidateDataAnnotations()` 启动时校验
- **非 ASP.NET**：引入 `Microsoft.Extensions.Hosting` 获得最佳体验
- **WPF/桌面**：通过 `IServiceScope` 手动管理作用域来使用 `IOptionsSnapshot`
- **自定义源**：实现 `IConfigurationSource` + `ConfigurationProvider`，在 `OnReload()` 中触发变更通知
