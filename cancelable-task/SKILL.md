---
name: cancelable-task
description: C# wrapper classes that convert Thread/Process into cancellable Task/Task<T> with CancellationToken support, timeout, IDisposable cleanup, and generic result types
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash]
---

# Cancellable Task Wrappers (Thread / Process → Task&lt;T&gt;)

将 `Thread` 和 `Process` 包装为可取消的 `Task` / `Task<T>`，支持 `CancellationToken`、超时、资源清理和泛型返回值。

> **原版代码**: <https://gist.github.com/BYJRK/29f10593cb6899294deb08dd972a6a27>  
> 原版仅支持 `Task`（无返回值），本 skill 在其基础上增加了 `Task<T>` 泛型支持、`IDisposable`、超时、进程输出收集等改进。

## 概述

当需要将**遗留同步代码**、**COM 互操作**、**CPU 密集型工作**或**外部进程**集成到 async/await 模型时，不能直接用 `Task.Run`（它只取消等待，不取消工作本身）。这套 wrapper 提供真正的取消能力：

| 类 | 包装对象 | 返回值 | 取消方式 |
|---|---------|--------|---------|
| `CancelableThreadTask` | `Action` | `Task` | `Thread.Interrupt` |
| `CancelableThreadTask<T>` | `Func<T>` | `Task<T>` | `Thread.Interrupt` |
| `CancelableProcessTask` | 外部进程 | `Task<ProcessResult>` | `Process.Kill` |
| `CancelableProcessTask<T>` | 外部进程 + 结果解析 | `Task<T>` | `Process.Kill` |

### 相比原版的关键改进

- **支持泛型返回值** — `Func<T>` / 进程输出解析
- **`IDisposable`** — 防止资源泄漏
- **`ProcessResult` 结构体** — 结构化封装进程输出（ExitCode、StdOut、StdError）
- **超时支持** — 内置 `timeout` 参数，通过 `CancellationTokenSource` 联动
- **启动失败处理** — 进程启动失败不再静默丢失
- **线程安全** — 完整的 Interlocked 状态机，防止重复启动和竞态条件

---

## 1. CancelableThreadTask（无返回值）

```csharp
public class CancelableThreadTask : IDisposable
{
    private Thread? _thread;
    private readonly Action _action;
    private CancellationTokenRegistration _ctr;
    private TaskCompletionSource? _tcs;
    private int _isRunning;
    private bool _disposed;

    public CancelableThreadTask(Action action)
    {
        _action = action ?? throw new ArgumentNullException(nameof(action));
    }

    /// <summary>
    /// 在线程上执行 action，支持取消和超时。
    /// </summary>
    /// <param name="token">取消令牌</param>
    /// <param name="timeout">超时时间，null 表示无超时</param>
    /// <returns>Task</returns>
    /// <exception cref="InvalidOperationException">任务已在运行</exception>
    /// <exception cref="ObjectDisposedException">实例已释放</exception>
    public Task RunAsync(CancellationToken token = default, TimeSpan? timeout = null)
    {
        ThrowIfDisposed();

        if (Interlocked.CompareExchange(ref _isRunning, 1, 0) == 1)
            throw new InvalidOperationException("Task is already running.");

        // 支持超时
        CancellationTokenSource? linkedCts = null;
        if (timeout.HasValue)
        {
            linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token);
            linkedCts.CancelAfter(timeout.Value);
            token = linkedCts.Token;
        }

        _tcs = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);

        _thread = new Thread(() =>
        {
            try
            {
                _action();
                _tcs.TrySetResult();
            }
            catch (Exception ex) when (ex is ThreadInterruptedException || token.IsCancellationRequested)
            {
                _tcs.TrySetCanceled(token);
            }
            catch (Exception ex)
            {
                _tcs.TrySetException(ex);
            }
            finally
            {
                Interlocked.Exchange(ref _isRunning, 0);
                linkedCts?.Dispose();
            }
        })
        {
            IsBackground = true,
            Name = $"CancelableThreadTask-{Guid.NewGuid():N}"
        };

        _ctr = token.Register(() =>
        {
            if (Interlocked.CompareExchange(ref _isRunning, 0, 1) == 1)
            {
                _thread.Interrupt();
                _tcs.TrySetCanceled(token);
            }
        });

        _thread.Start();
        return _tcs.Task;
    }

    private void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(CancelableThreadTask));
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _ctr.Dispose();
    }
}
```

---

## 2. CancelableThreadTask&lt;T&gt;（泛型返回值）

```csharp
public class CancelableThreadTask<T> : IDisposable
{
    private Thread? _thread;
    private readonly Func<T> _func;
    private CancellationTokenRegistration _ctr;
    private TaskCompletionSource<T>? _tcs;
    private int _isRunning;
    private bool _disposed;

    public CancelableThreadTask(Func<T> func)
    {
        _func = func ?? throw new ArgumentNullException(nameof(func));
    }

    /// <summary>
    /// 在线程上执行 func，返回 Task&lt;T&gt;，支持取消和超时。
    /// </summary>
    public Task<T> RunAsync(CancellationToken token = default, TimeSpan? timeout = null)
    {
        ThrowIfDisposed();

        if (Interlocked.CompareExchange(ref _isRunning, 1, 0) == 1)
            throw new InvalidOperationException("Task is already running.");

        CancellationTokenSource? linkedCts = null;
        if (timeout.HasValue)
        {
            linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token);
            linkedCts.CancelAfter(timeout.Value);
            token = linkedCts.Token;
        }

        _tcs = new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously);

        _thread = new Thread(() =>
        {
            try
            {
                T result = _func();
                _tcs.TrySetResult(result);
            }
            catch (Exception ex) when (ex is ThreadInterruptedException || token.IsCancellationRequested)
            {
                _tcs.TrySetCanceled(token);
            }
            catch (Exception ex)
            {
                _tcs.TrySetException(ex);
            }
            finally
            {
                Interlocked.Exchange(ref _isRunning, 0);
                linkedCts?.Dispose();
            }
        })
        {
            IsBackground = true,
            Name = $"CancelableThreadTask-{Guid.NewGuid():N}"
        };

        _ctr = token.Register(() =>
        {
            if (Interlocked.CompareExchange(ref _isRunning, 0, 1) == 1)
            {
                _thread.Interrupt();
                _tcs.TrySetCanceled(token);
            }
        });

        _thread.Start();
        return _tcs.Task;
    }

    private void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(CancelableThreadTask<T>));
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _ctr.Dispose();
    }
}
```

---

## 3. ProcessResult 结构体

```csharp
/// <summary>
/// 进程执行结果。
/// </summary>
public readonly struct ProcessResult
{
    public int ExitCode { get; }
    public string StdOut { get; }
    public string StdError { get; }

    public ProcessResult(int exitCode, string stdOut, string stdError)
    {
        ExitCode = exitCode;
        StdOut = stdOut;
        StdError = stdError;
    }

    public bool Success => ExitCode == 0;

    public void Deconstruct(out int exitCode, out string stdOut, out string stdError)
    {
        exitCode = ExitCode;
        stdOut = StdOut;
        stdError = StdError;
    }

    public override string ToString()
        => $"ExitCode={ExitCode}, StdOut={StdOut?.Length ?? 0} chars, StdError={StdError?.Length ?? 0} chars";
}
```

---

## 4. CancelableProcessTask（返回 ProcessResult）

```csharp
public class CancelableProcessTask : IDisposable
{
    private readonly string _fileName;
    private readonly string _arguments;
    private Process? _process;
    private CancellationTokenRegistration _ctr;
    private TaskCompletionSource<ProcessResult>? _tcs;
    private int _isRunning;
    private bool _disposed;

    public CancelableProcessTask(string fileName, string arguments)
    {
        _fileName = fileName ?? throw new ArgumentNullException(nameof(fileName));
        _arguments = arguments ?? "";
    }

    /// <summary>
    /// 启动进程，返回 Task&lt;ProcessResult&gt;，支持取消和超时。
    /// </summary>
    public Task<ProcessResult> RunAsync(CancellationToken token = default, TimeSpan? timeout = null)
    {
        ThrowIfDisposed();

        if (Interlocked.CompareExchange(ref _isRunning, 1, 0) == 1)
            throw new InvalidOperationException("Process is already running.");

        CancellationTokenSource? linkedCts = null;
        if (timeout.HasValue)
        {
            linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token);
            linkedCts.CancelAfter(timeout.Value);
            token = linkedCts.Token;
        }

        _tcs = new TaskCompletionSource<ProcessResult>(TaskCreationOptions.RunContinuationsAsynchronously);

        try
        {
            _process = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = _fileName,
                    Arguments = _arguments,
                    UseShellExecute = false,
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    CreateNoWindow = true,
                },
                EnableRaisingEvents = true,
            };

            // 收集输出（异步读取防止死锁）
            var stdOutBuilder = new StringBuilder();
            var stdErrorBuilder = new StringBuilder();

            _process.OutputDataReceived += (_, e) => stdOutBuilder.AppendLine(e.Data);
            _process.ErrorDataReceived += (_, e) => stdErrorBuilder.AppendLine(e.Data);

            _process.Exited += (_, _) =>
            {
                // 等待异步输出完成
                _process.WaitForExit();

                var result = new ProcessResult(
                    _process.ExitCode,
                    stdOutBuilder.ToString().TrimEnd(),
                    stdErrorBuilder.ToString().TrimEnd());

                if (_process.ExitCode == 0)
                    _tcs.TrySetResult(result);
                else if (token.IsCancellationRequested)
                    _tcs.TrySetCanceled(token);
                else
                    _tcs.TrySetException(
                        new InvalidOperationException(
                            $"Process exited with code {_process.ExitCode}. Stderr: {result.StdError}"));
            };

            if (!_process.Start())
            {
                _tcs.TrySetException(new InvalidOperationException($"Failed to start process: {_fileName}"));
                Interlocked.Exchange(ref _isRunning, 0);
                linkedCts?.Dispose();
                return _tcs.Task;
            }

            _process.BeginOutputReadLine();
            _process.BeginErrorReadLine();
        }
        catch (Exception ex)
        {
            _tcs.TrySetException(ex);
            Interlocked.Exchange(ref _isRunning, 0);
            linkedCts?.Dispose();
            return _tcs.Task;
        }

        _ctr = token.Register(() =>
        {
            if (Interlocked.CompareExchange(ref _isRunning, 0, 1) == 1)
            {
                try { _process?.Kill(entireProcessTree: true); }
                catch (InvalidOperationException) { /* 进程已退出 */ }
                _tcs.TrySetCanceled(token);
            }
        });

        return _tcs.Task;
    }

    private void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(CancelableProcessTask));
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _ctr.Dispose();
        _process?.Dispose();
    }
}
```

---

## 5. CancelableProcessTask&lt;T&gt;（泛型，带结果选择器）

```csharp
public class CancelableProcessTask<T> : IDisposable
{
    private readonly string _fileName;
    private readonly string _arguments;
    private readonly Func<ProcessResult, T> _resultSelector;
    private Process? _process;
    private CancellationTokenRegistration _ctr;
    private TaskCompletionSource<T>? _tcs;
    private int _isRunning;
    private bool _disposed;

    /// <summary>
    /// </summary>
    /// <param name="fileName">可执行文件路径</param>
    /// <param name="arguments">命令行参数</param>
    /// <param name="resultSelector">将 ProcessResult 映射为 T，返回结果或将错误包装为异常</param>
    public CancelableProcessTask(string fileName, string arguments, Func<ProcessResult, T> resultSelector)
    {
        _fileName = fileName ?? throw new ArgumentNullException(nameof(fileName));
        _arguments = arguments ?? "";
        _resultSelector = resultSelector ?? throw new ArgumentNullException(nameof(resultSelector));
    }

    /// <summary>
    /// 启动进程，返回 Task&lt;T&gt;，支持取消和超时。
    /// </summary>
    public Task<T> RunAsync(CancellationToken token = default, TimeSpan? timeout = null)
    {
        ThrowIfDisposed();

        if (Interlocked.CompareExchange(ref _isRunning, 1, 0) == 1)
            throw new InvalidOperationException("Process is already running.");

        CancellationTokenSource? linkedCts = null;
        if (timeout.HasValue)
        {
            linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token);
            linkedCts.CancelAfter(timeout.Value);
            token = linkedCts.Token;
        }

        _tcs = new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously);

        try
        {
            _process = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = _fileName,
                    Arguments = _arguments,
                    UseShellExecute = false,
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    CreateNoWindow = true,
                },
                EnableRaisingEvents = true,
            };

            var stdOutBuilder = new StringBuilder();
            var stdErrorBuilder = new StringBuilder();

            _process.OutputDataReceived += (_, e) => stdOutBuilder.AppendLine(e.Data);
            _process.ErrorDataReceived += (_, e) => stdErrorBuilder.AppendLine(e.Data);

            _process.Exited += (_, _) =>
            {
                _process.WaitForExit();

                var raw = new ProcessResult(
                    _process.ExitCode,
                    stdOutBuilder.ToString().TrimEnd(),
                    stdErrorBuilder.ToString().TrimEnd());

                try
                {
                    T mapped = _resultSelector(raw);
                    _tcs.TrySetResult(mapped);
                }
                catch (Exception ex)
                {
                    _tcs.TrySetException(ex);
                }
            };

            if (!_process.Start())
            {
                _tcs.TrySetException(new InvalidOperationException($"Failed to start process: {_fileName}"));
                Interlocked.Exchange(ref _isRunning, 0);
                linkedCts?.Dispose();
                return _tcs.Task;
            }

            _process.BeginOutputReadLine();
            _process.BeginErrorReadLine();
        }
        catch (Exception ex)
        {
            _tcs.TrySetException(ex);
            Interlocked.Exchange(ref _isRunning, 0);
            linkedCts?.Dispose();
            return _tcs.Task;
        }

        _ctr = token.Register(() =>
        {
            if (Interlocked.CompareExchange(ref _isRunning, 0, 1) == 1)
            {
                try { _process?.Kill(entireProcessTree: true); }
                catch (InvalidOperationException) { }
                _tcs.TrySetCanceled(token);
            }
        });

        return _tcs.Task;
    }

    private void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(CancelableProcessTask<T>));
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _ctr.Dispose();
        _process?.Dispose();
    }
}
```

---

## 使用示例

### 线程示例

```csharp
// 无返回值
using var task = new CancelableThreadTask(() =>
{
    Thread.Sleep(5000);  // 模拟耗时同步操作
    File.WriteAllText("result.txt", "done");
});

var cts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
try
{
    await task.RunAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("线程任务已取消");
}

// 带返回值
using var task2 = new CancelableThreadTask<int>(() =>
{
    Thread.Sleep(2000);
    return ComputeHeavyResult(); // 返回 int
});

int result = await task2.RunAsync(CancellationToken.None, TimeSpan.FromSeconds(10));
```

### 进程示例

```csharp
// 返回 ProcessResult
using var proc = new CancelableProcessTask("git", "status --porcelain");
var result = await proc.RunAsync(CancellationToken.None, TimeSpan.FromSeconds(30));

if (result.Success)
    Console.WriteLine($"Changed files:\n{result.StdOut}");
else
    Console.Error.WriteLine($"Git failed:\n{result.StdError}");

// 或解构
var (exitCode, stdout, stderr) = result;

// 泛型：解析 JSON 输出
using var proc2 = new CancelableProcessTask<MyConfig>(
    "my-cli", "--output json",
    pr => JsonSerializer.Deserialize<MyConfig>(pr.StdOut)
        ?? throw new InvalidOperationException("Null output"));

MyConfig config = await proc2.RunAsync(cts.Token);
```

### 超时方式对比

```csharp
// 方式1: 传入 timeout 参数（推荐）
await proc.RunAsync(token, TimeSpan.FromSeconds(30));

// 方式2: 使用 CancellationTokenSource
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
await proc.RunAsync(cts.Token);

// 方式3: 外部取消 + 超时联动（两者都生效）
var cts2 = new CancellationTokenSource();
await proc.RunAsync(cts2.Token, TimeSpan.FromSeconds(30));
// 任意一个触发即取消
```

## 关键约定与注意事项

- **单次运行** — `RunAsync` 只能调用一次（Interlocked 状态机保护），重复调用抛出 `InvalidOperationException`
- **必须 Dispose** — 或用 `using` 语句，否则 `CancellationTokenRegistration` 会泄漏到 `CancellationTokenSource` 的链表
- **`Thread.Interrupt` 的局限性** — 只在目标线程处于 `WaitSleepJoin` 状态时生效；CPU 密集型循环需自行检查 `token.IsCancellationRequested`
- **进程取消** — 使用 `Kill(entireProcessTree: true)` 强制终止整个进程树
- **输出缓冲区死锁** — 必须使用 `BeginOutputReadLine/BeginErrorReadLine` 异步读取，否则缓冲区满会导致进程挂起
- **`TaskCreationOptions.RunContinuationsAsynchronously`** — 防止 `SetResult`/`SetCanceled` 时同步执行延续，避免死锁
- **`ProcessResult` 是不可变值类型** — 避免引用类型的分配和空检查
- **进程使用 `WaitForExit` 后** — 确保 `BeginOutputReadLine` 的异步回调已完成，输出完整收集

## 适用场景

- 遗留代码集成（COM 互操作、POSIX 信号量等不支持 `async` 的操作）
- 外部命令行工具调用（ffmpeg、git、自定义 CLI）
- 需要真正取消的 CPU 密集型同步工作（`Task.Run` + `token` 仅取消等待，不中断工作）
- 需要进程输出结构化解析的场景（JSON 输出、自定义格式）
