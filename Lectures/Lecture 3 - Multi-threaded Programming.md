# Lecture 3 — Multi-threaded Programming (C#)

**(Preparing for GUI Programming: WinForms/WPF)**

**Course:** Application Development 1 (Desktop)  
**Goal today:** Understand _why GUIs freeze_, how to keep apps responsive, and how to do background work safely.

---

## Learning outcomes

By the end of this lecture, you can:

- Explain the difference between **Thread**, **Task**, and **async/await**
- Identify **UI thread blocking** (the #1 GUI bug)
- Use `Task.Run` for CPU-bound work and `await` for I/O-bound work
- Cancel long-running operations with `CancellationToken`
- Report progress safely with `IProgress<T>`
- Avoid race conditions with `lock` / `Interlocked`
- Understand how GUI apps update UI from background work (`Invoke` / `Dispatcher`)

---

## 1) The big idea: the UI thread

GUI frameworks (WinForms/WPF) have a **single UI thread** responsible for:

- Handling button clicks, typing, window dragging, painting
- Running event handlers like `Button_Click`

If you do heavy work on that UI thread:

- the app becomes unresponsive
- you get “Not responding”
- animation/spinners stop
- clicks feel “dead”

**Rule (non-negotiable):**  
✅ Keep the UI thread free. Do heavy work on background threads (Tasks) and _report results back_.

---

## 2) Thread vs Task vs async/await

### `Thread`

- Lower-level primitive
- You manually create/manage a thread
- More control, more pain

### `Task`

- High-level, modern approach
- Uses the **ThreadPool** by default
- Better for most app scenarios

### `async/await`

- A syntax feature that makes asynchronous code readable
- **Does not automatically create threads**
- Most often used for I/O operations (files, web, database) to avoid blocking

**Mental model:**

- **CPU-bound work** (heavy calculations): use `Task.Run(...)` to move it off UI thread
- **I/O-bound work** (file/network/database): use `await SomethingAsync()` (no need for Task.Run)

---

## 3) Demo: “Freezing” (what happens in a GUI)

In a GUI, if you did this inside a button click, the UI would freeze.

```csharp
using System;

class Program
{
    static void Main()
    {
        Console.WriteLine("Starting heavy work...");
        HeavyWorkBlocking(); // Imagine this is a button click handler
        Console.WriteLine("Done!");
    }

    static void HeavyWorkBlocking()
    {
        // Simulate expensive work
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine($"Working... {i + 1}/5");
            Thread.Sleep(1000); // blocks the current thread
        }
    }
}
```

**Key point:** `Thread.Sleep` blocks the thread entirely. In GUI: that thread is the UI thread → frozen app.

---

## 4) Demo: Keep the “UI” responsive using Task + async/await

This is the pattern you’ll use constantly in WinForms/WPF:

- Start work on background thread
- Keep the main thread responsive
- Await completion

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        Console.WriteLine("Starting heavy work in background...");

        // Start background work
        Task work = Task.Run(() => HeavyWorkCpuBound());

        // Simulate UI responsiveness with a spinner while work runs
        char[] spinner = { '|', '/', '-', '\\' };
        int idx = 0;

        while (!work.IsCompleted)
        {
            Console.Write($"\rWorking {spinner[idx++ % spinner.Length]} ");
            await Task.Delay(100); // DOES NOT block thread like Sleep
        }

        await work; // observe exceptions if any
        Console.WriteLine("\rDone!            ");
    }

    static void HeavyWorkCpuBound()
    {
        for (int i = 0; i < 5; i++)
        {
            Thread.Sleep(1000); // safe here because it's on a background thread
        }
    }
}
```

**In a GUI:** the spinner is your UI repainting + responding to clicks while work runs.

---

## 5) Cancellation (critical for good desktop apps)

In GUI apps, users expect a **Cancel** button.
Cancellation in .NET uses `CancellationToken`.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        using var cts = new CancellationTokenSource();

        Console.WriteLine("Press 'c' to cancel.");

        Task work = Task.Run(() => LongWork(cts.Token), cts.Token);

        // Listen for cancel key
        while (!work.IsCompleted)
        {
            if (Console.KeyAvailable && Console.ReadKey(true).KeyChar == 'c')
            {
                cts.Cancel();
                Console.WriteLine("Cancel requested...");
                break;
            }

            await Task.Delay(50);
        }

        try
        {
            await work;
            Console.WriteLine("Work completed successfully.");
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("Work was cancelled.");
        }
    }

    static void LongWork(CancellationToken token)
    {
        for (int i = 1; i <= 100; i++)
        {
            token.ThrowIfCancellationRequested();
            Thread.Sleep(50); // simulate work
        }
    }
}
```

**GUI mapping:**

- Cancel button calls `cts.Cancel()`
- Background operation checks token regularly

---

## 6) Progress reporting (for progress bars)

Don’t update UI controls from background threads directly.
Use `IProgress<T>` which posts updates back safely.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        var progress = new Progress<int>(percent =>
        {
            Console.WriteLine($"Progress: {percent}%");
        });

        await Task.Run(() => DoWorkWithProgress(progress));
        Console.WriteLine("Done.");
    }

    static void DoWorkWithProgress(IProgress<int> progress)
    {
        for (int i = 1; i <= 10; i++)
        {
            Thread.Sleep(300);
            progress.Report(i * 10);
        }
    }
}
```

**GUI mapping:**

- `progress.Report(x)` updates a ProgressBar value safely (when wired correctly)
- In WinForms/WPF you often create `Progress<int>` on the UI thread so its callback runs on UI thread

---

## 7) Race conditions (the silent killer)

A race condition happens when multiple threads access shared data and results depend on timing.

### Bad example (wrong result)

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        int counter = 0;

        Parallel.For(0, 1_000_000, i =>
        {
            counter++; // NOT thread-safe
        });

        Console.WriteLine(counter); // often < 1,000,000
    }
}
```

### Fix 1: `lock`

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        int counter = 0;
        object gate = new object();

        Parallel.For(0, 1_000_000, i =>
        {
            lock (gate)
            {
                counter++;
            }
        });

        Console.WriteLine(counter); // correct
    }
}
```

### Fix 2: `Interlocked` (fast for simple counters)

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        int counter = 0;

        Parallel.For(0, 1_000_000, i =>
        {
            Interlocked.Increment(ref counter);
        });

        Console.WriteLine(counter); // correct
    }
}
```

**Rule:** shared mutable state requires synchronization.

---

## 8) What GUIs require: “only the UI thread may touch UI controls”

### WinForms (concept)

If a background task tries:

```csharp
myLabel.Text = "Done";
```

…it can throw an exception or behave unpredictably.

Correct concept is: marshal back to UI thread using `Invoke`:

```csharp
// WinForms concept (not runnable in console)
this.Invoke(() => myLabel.Text = "Done");
```

### WPF (concept)

Use `Dispatcher`:

```csharp
// WPF concept (not runnable in console)
Dispatcher.Invoke(() => MyLabel.Content = "Done");
```

### The modern approach: `async` event handlers

In GUIs, you typically do:

- Event handler stays responsive
- Await I/O or Task.Run for CPU work
- Update UI after `await` (you’re back on UI thread in typical GUI contexts)

Pseudo-pattern:

```csharp
// WinForms/WPF pattern (concept)
async void Button_Click(object sender, EventArgs e)
{
    button.Enabled = false;

    try
    {
        var result = await Task.Run(() => ExpensiveCpuWork());
        label.Text = result; // safe after await (typical UI context)
    }
    finally
    {
        button.Enabled = true;
    }
}
```

---

## 9) Practical rules for GUI programming (memorize these)

1. **Never block the UI thread**
   - avoid `Thread.Sleep`, heavy loops, large file reads synchronously in UI events

2. **I/O-bound work**
   - use `await File.ReadAllTextAsync(...)`, `await ...Async()`

3. **CPU-bound work**
   - use `await Task.Run(() => ...)`

4. **Cancellation is part of UX**
   - every long action should be cancellable

5. **Progress makes apps feel fast**
   - use `IProgress<T>` rather than touching UI from background threads

6. **Shared data needs protection**
   - use `lock`, `Interlocked`, or concurrent collections

---

## 10) Mini-exercise (in class, 10–15 min)

Build a console “download simulator”:

- runs work in background
- prints progress every 10%
- supports cancel with key press
- never freezes input loop

(You now have all building blocks above.)
