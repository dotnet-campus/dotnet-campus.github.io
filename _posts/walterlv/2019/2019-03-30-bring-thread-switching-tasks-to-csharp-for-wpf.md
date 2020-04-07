---
title: "将 C++/WinRT 中的线程切换体验带到 C# 中来（WPF 版本）"
publishDate: 2019-03-30 08:24:45 +0800
date: 2019-03-30 09:13:12 +0800
categories: dotnet csharp wpf
position: play
---

如果你要在 WPF 程序中使用线程池完成一个特殊的任务，那么使用 .NET 的 API `Task.Run` 并传入一个 Lambda 表达式可以完成。不过，使用 Lambda 表达式会带来变量捕获的一些问题，比如说你需要区分一个变量作用于是在 Lambda 表达式中，还是当前上下文全局（被 Lambda 表达式捕获到的变量）。然后，在静态分析的时候，也难以知道此 Lambda 表达式在整个方法中的执行先后顺序，不利于分析潜在的 Bug。

在使用 `async`/`await` 关键字编写异步代码的时候，虽然说实质上也是捕获变量，但这时没有显式写一个 Lambda 表达式，所有的变量都是被隐式捕获的变量，写起来就像在一个同步方法一样，便于理解。

---

<div id="toc"></div>

## C++/WinRT

以下 C++/WinRT 的代码来自 Raymond Chen 的示例代码。Raymond Chen 写了一个 UWP 的版本用于模仿 C++/WinRT 的线程切换效果。在看他编写的 UWP 版本之前我也思考了可以如何实现一个 .NET / WPF 的版本，然后成功做出了这样的效果。

Raymond Chen 的版本可以参见：[C++/WinRT envy: Bringing thread switching tasks to C# (UWP edition) - The Old New Thing](https://devblogs.microsoft.com/oldnewthing/20190328-00/?p=102368)。

```cpp
winrt::fire_and_forget MyPage::Button_Click()
{
  // We start on a UI thread.
  auto lifetime = get_strong();

  // Get the control's value from the UI thread.
  auto v = SomeControl().Value();

  // Move to a background thread.
  co_await winrt::resume_background();

  // Do the computation on a background thread.
  auto result1 = Compute1(v);
  auto other = co_await ContactWebServiceAsync();
  auto result2 = Compute2(result1, other);

  // Return to the UI thread to provide an interim update.
  co_await winrt::resume_foreground(Dispatcher());

  // Back on the UI thread: We can update UI elements.
  TextBlock1().Text(result1);
  TextBlock2().Text(result2);

  // Back to the background thread to do more computations.
  co_await winrt::resume_background();

  auto extra = co_await GetExtraDataAsync();
  auto result3 = Compute3(result1, result2, extra);

  // Return to the UI thread to provide a final update.
  co_await winrt::resume_foreground(Dispatcher());

  // Update the UI one last time.
  TextBlock3().Text(result3);
}
```

可以看到，使用 `co_await winrt::resume_background();` 可以将线程切换至线程池，使用 `co_await winrt::resume_foreground(Dispatcher());` 可以将线程切换至 UI。

也许你会觉得这样没什么好处，因为 C#/.NET 的版本里面 Lambda 表达式一样可以这么做：

```csharp
await Task.Run(() =>
{
    // 这里的代码会在线程池执行。
});
// 这里的代码会回到 UI 线程执行。
```

但是，现在我们给出这样的写法：

```cpp
// 仅在某些特定的情况下才使用线程池执行，而其他情况依然在主线程执行 DoSomething()。
if (condition) {
  co_await winrt::resume_background();
}

DoSomething();
```

你就会发现 Lambda 的版本变得很不好理解了。

## C# / .NET / WPF 版本

我们现在编写一个自己的 Awaiter 来实现这样的线程上下文切换。

关于如何编写一个 Awaiter，可以阅读我的其他博客：

- [定义一组抽象的 Awaiter 的实现接口，你下次写自己的 await 可等待对象时将更加方便 - 吕毅](/post/abstract-awaitable-and-awaiter)
- [.NET 中什么样的类是可使用 await 异步等待的？ - 吕毅](/post/what-is-an-awaiter)
- [.NET 除了用 Task 之外，如何自己写一个可以 await 的对象？ - 吕毅](/post/understand-and-write-custom-awaiter)

这里，我直接贴出我编写的 `DispatcherSwitcher` 类的全部源码。

```csharp
using System;
using System.Runtime.CompilerServices;
using System.Threading.Tasks;
using System.Windows.Threading;

namespace Walterlv.ThreadSwitchingTasks
{
    public static class DispatcherSwitcher
    {
        public static ThreadPoolAwaiter ResumeBackground() => new ThreadPoolAwaiter();

        public static ThreadPoolAwaiter ResumeBackground(this Dispatcher dispatcher)
            => new ThreadPoolAwaiter();

        public static DispatcherAwaiter ResumeForeground(this Dispatcher dispatcher) =>
            new DispatcherAwaiter(dispatcher);

        public class ThreadPoolAwaiter : INotifyCompletion
        {
            public void OnCompleted(Action continuation)
            {
                Task.Run(() =>
                {
                    IsCompleted = true;
                    continuation();
                });
            }

            public bool IsCompleted { get; private set; }

            public void GetResult()
            {
            }

            public ThreadPoolAwaiter GetAwaiter() => this;
        }

        public class DispatcherAwaiter : INotifyCompletion
        {
            private readonly Dispatcher _dispatcher;

            public DispatcherAwaiter(Dispatcher dispatcher) => _dispatcher = dispatcher;

            public void OnCompleted(Action continuation)
            {
                _dispatcher.InvokeAsync(() =>
                {
                    IsCompleted = true;
                    continuation();
                });
            }

            public bool IsCompleted { get; private set; }

            public void GetResult()
            {
            }

            public DispatcherAwaiter GetAwaiter() => this;
        }
    }
}
```

Raymond Chen 取的类名是 `ThreadSwitcher`，不过我认为可能 `Dispatcher` 在 WPF 中更能体现其线程切换的含义。

于是，我们来做一个试验。以下代码在 MainWindow.xaml.cs 里面，如果你使用 Visual Studio 创建一个 WPF 的空项目的话是可以找到的。随便放一个 Button 添加事件处理函数。

```csharp
private async void DemoButton_Click(object sender, RoutedEventArgs e)
{
    var id0 = Thread.CurrentThread.ManagedThreadId;

    await Dispatcher.ResumeBackground();

    var id1 = Thread.CurrentThread.ManagedThreadId;

    await Dispatcher.ResumeForeground();

    var id2 = Thread.CurrentThread.ManagedThreadId;
}
```

id0 和 id2 在主线程上，id1 是线程池中的一个线程。

这样，我们便可以在一个上下文中进行线程切换了，而不需要使用 `Task.Run` 通过一个 Lambda 表达式来完成这样的任务。

现在，这种按照某些特定条件才切换到后台线程执行的代码就很容易写出来了。

```csharp
// 仅在某些特定的情况下才使用线程池执行，而其他情况依然在主线程执行 DoSomething()。
if (condition)
{
    await Dispatcher.ResumeBackground();
}

DoSomething();
```

## Raymond Chen 的版本

Raymond Chen 后来在另一篇博客中也编写了一份 WPF / Windows Forms 的线程切换版本。请点击下方的链接跳转至原文阅读：

- [C++/WinRT envy: Bringing thread switching tasks to C# (WPF and WinForms edition) - The Old New Thing](https://devblogs.microsoft.com/oldnewthing/20190329-00/?p=102373)

我在为他的代码添加了所有的注释后，贴在了下面：

```csharp
using System;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Windows.Forms;
using System.Windows.Threading;

namespace Walterlv.Windows.Threading
{
    /// <summary>
    /// 提供类似于 WinRT 中的线程切换体验。
    /// </summary>
    /// <remarks>
    /// https://devblogs.microsoft.com/oldnewthing/20190329-00/?p=102373
    /// https://blog.walterlv.com/post/bring-thread-switching-tasks-to-csharp-for-wpf.html
    /// </remarks>
    public class ThreadSwitcher
    {
        /// <summary>
        /// 将当前的异步等待上下文切换到 WPF 的 UI 线程中继续执行。
        /// </summary>
        /// <param name="dispatcher">WPF 一个 UI 线程的调度器。</param>
        /// <returns>一个可等待对象，使用 await 等待此对象可以使后续任务切换到 UI 线程执行。</returns>
        public static DispatcherThreadSwitcher ResumeForegroundAsync(Dispatcher dispatcher) =>
            new DispatcherThreadSwitcher(dispatcher);

        /// <summary>
        /// 将当前的异步等待上下文切换到 Windows Forms 的 UI 线程中继续执行。
        /// </summary>
        /// <param name="control">Windows Forms 的一个控件。</param>
        /// <returns>一个可等待对象，使用 await 等待此对象可以使后续任务切换到 UI 线程执行。</returns>
        public static ControlThreadSwitcher ResumeForegroundAsync(Control control) =>
            new ControlThreadSwitcher(control);

        /// <summary>
        /// 将当前的异步等待上下文切换到线程池中继续执行。
        /// </summary>
        /// <returns>一个可等待对象，使用 await 等待此对象可以使后续的任务切换到线程池执行。</returns>
        public static ThreadPoolThreadSwitcher ResumeBackgroundAsync() =>
            new ThreadPoolThreadSwitcher();
    }

    /// <summary>
    /// 提供一个可切换到 WPF 的 UI 线程执行上下文的可等待对象。
    /// </summary>
    public struct DispatcherThreadSwitcher : INotifyCompletion
    {
        internal DispatcherThreadSwitcher(Dispatcher dispatcher) =>
            _dispatcher = dispatcher;

        /// <summary>
        /// 当使用 await 关键字异步等待此对象时，将调用此方法返回一个可等待对象。
        /// </summary>
        public DispatcherThreadSwitcher GetAwaiter() => this;

        /// <summary>
        /// 获取一个值，该值指示是否已完成线程池到 WPF UI 线程的切换。
        /// </summary>
        public bool IsCompleted => _dispatcher.CheckAccess();

        /// <summary>
        /// 由于进行线程的上下文切换必须使用 await 关键字，所以不支持调用同步的 <see cref="GetResult"/> 方法。
        /// </summary>
        public void GetResult()
        {
        }

        /// <summary>
        /// 当异步状态机中的前一个任务结束后，将调用此方法继续下一个任务。在此可等待对象中，指的是切换到 WPF 的 UI 线程。
        /// </summary>
        /// <param name="continuation">将异步状态机推进到下一个异步状态。</param>
        public void OnCompleted(Action continuation) => _dispatcher.BeginInvoke(continuation);

        private readonly Dispatcher _dispatcher;
    }

    /// <summary>
    /// 提供一个可切换到 Windows Forms 的 UI 线程执行上下文的可等待对象。
    /// </summary>
    public struct ControlThreadSwitcher : INotifyCompletion
    {
        internal ControlThreadSwitcher(Control control) =>
            _control = control;

        /// <summary>
        /// 当使用 await 关键字异步等待此对象时，将调用此方法返回一个可等待对象。
        /// </summary>
        public ControlThreadSwitcher GetAwaiter() => this;

        /// <summary>
        /// 获取一个值，该值指示是否已完成线程池到 Windows Forms UI 线程的切换。
        /// </summary>
        public bool IsCompleted => !_control.InvokeRequired;

        /// <summary>
        /// 由于进行线程的上下文切换必须使用 await 关键字，所以不支持调用同步的 <see cref="GetResult"/> 方法。
        /// </summary>
        public void GetResult()
        {
        }

        /// <summary>
        /// 当异步状态机中的前一个任务结束后，将调用此方法继续下一个任务。在此可等待对象中，指的是切换到 Windows Forms 的 UI 线程。
        /// </summary>
        /// <param name="continuation">将异步状态机推进到下一个异步状态。</param>
        public void OnCompleted(Action continuation) => _control.BeginInvoke(continuation);

        private readonly Control _control;
    }

    /// <summary>
    /// 提供一个可切换到线程池执行上下文的可等待对象。
    /// </summary>
    public struct ThreadPoolThreadSwitcher : INotifyCompletion
    {
        /// <summary>
        /// 当使用 await 关键字异步等待此对象时，将调用此方法返回一个可等待对象。
        /// </summary>
        public ThreadPoolThreadSwitcher GetAwaiter() => this;

        /// <summary>
        /// 获取一个值，该值指示是否已完成 UI 线程到线程池的切换。
        /// </summary>
        public bool IsCompleted => SynchronizationContext.Current == null;

        /// <summary>
        /// 由于进行线程的上下文切换必须使用 await 关键字，所以不支持调用同步的 <see cref="GetResult"/> 方法。
        /// </summary>
        public void GetResult()
        {
        }

        /// <summary>
        /// 当异步状态机中的前一个任务结束后，将调用此方法继续下一个任务。在此可等待对象中，指的是切换到线程池中。
        /// </summary>
        /// <param name="continuation">将异步状态机推进到下一个异步状态。</param>
        public void OnCompleted(Action continuation) => ThreadPool.QueueUserWorkItem(_ => continuation());
    }
}
```

---

**参考资料**

- [C++/WinRT envy: Bringing thread switching tasks to C# (UWP edition) - The Old New Thing](https://devblogs.microsoft.com/oldnewthing/20190328-00/?p=102368)
- [C++/WinRT envy: Bringing thread switching tasks to C# (WPF and WinForms edition) - The Old New Thing](https://devblogs.microsoft.com/oldnewthing/20190329-00/?p=102373)
