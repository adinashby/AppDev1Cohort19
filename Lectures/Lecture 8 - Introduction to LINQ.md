# Lecture 8 — Introduction to LINQ (C#)

**Course:** Application Development 1 (Desktop)  
**Goal today:** Learn LINQ so you can filter/sort/group data in your WPF apps (DataGrid filtering, searching, reporting) with clean, readable code.

---

## Learning outcomes

By the end of this lecture, you can:

- Explain what LINQ is and why it matters for desktop apps
- Use the core LINQ operators: `Where`, `Select`, `OrderBy`, `ThenBy`, `Take`, `Skip`
- Use aggregations: `Count`, `Sum`, `Average`, `Min`, `Max`
- Use `Any`, `All`, `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`
- Group data with `GroupBy`
- Understand deferred execution vs immediate execution (`ToList`, `ToArray`)
- Avoid common LINQ mistakes (multiple enumeration, nulls, `Single` traps)
- Apply LINQ to real app scenarios (search/filter in a DataGrid)

---

## 1) What is LINQ?

**LINQ** = _Language Integrated Query_  
It lets you query collections (arrays, lists, databases) using a consistent style.

LINQ is used everywhere in desktop apps:

- filtering items in a `DataGrid`
- searching by text
- sorting by columns
- generating summaries (“How many completed tasks?”)
- grouping reports (“Tasks per user”, “Orders per month”)

---

## 2) Setup: sample model + data (copy/paste)

Create a quick model for examples:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class TaskItem
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public bool Completed { get; set; }
    public int UserId { get; set; }
    public DateTime CreatedAt { get; set; }
}

var tasks = new List<TaskItem>
{
    new TaskItem { Id = 1, Title = "Buy milk", Completed = true,  UserId = 1, CreatedAt = DateTime.Now.AddDays(-2) },
    new TaskItem { Id = 2, Title = "Study LINQ", Completed = false, UserId = 1, CreatedAt = DateTime.Now.AddDays(-1) },
    new TaskItem { Id = 3, Title = "Finish Lab 3", Completed = false, UserId = 2, CreatedAt = DateTime.Now.AddDays(-4) },
    new TaskItem { Id = 4, Title = "Clean desk", Completed = true,  UserId = 2, CreatedAt = DateTime.Now.AddDays(-10) },
    new TaskItem { Id = 5, Title = "Email teacher", Completed = false, UserId = 1, CreatedAt = DateTime.Now.AddDays(-7) },
};
```

---

## 3) Two LINQ styles: Method syntax vs Query syntax

### Method syntax (most common in apps)

```csharp
var incomplete = tasks.Where(t => !t.Completed).ToList();
```

### Query syntax (looks like SQL)

```csharp
var incomplete =
    (from t in tasks
     where !t.Completed
     select t).ToList();
```

We’ll mostly use **method syntax** in this course.

---

## 4) Filtering with `Where`

```csharp
var completedTasks = tasks.Where(t => t.Completed).ToList();
var user1Tasks = tasks.Where(t => t.UserId == 1).ToList();
var containsLab = tasks.Where(t => t.Title.Contains("Lab", StringComparison.OrdinalIgnoreCase)).ToList();
```

**Important:** `Where` does not change the original list. It creates a new filtered sequence.

---

## 5) Projection with `Select` (transform items)

### Select only titles

```csharp
var titles = tasks.Select(t => t.Title).ToList();
```

### Select into an “anonymous object” (great for quick UI display/debug)

```csharp
var mini =
    tasks.Select(t => new
    {
        t.Id,
        t.Title,
        Status = t.Completed ? "Done" : "Pending"
    }).ToList();
```

---

## 6) Sorting with `OrderBy` / `ThenBy`

```csharp
var byTitle = tasks.OrderBy(t => t.Title).ToList();

var byCompletedThenDate =
    tasks.OrderBy(t => t.Completed)          // false before true
         .ThenByDescending(t => t.CreatedAt) // newest first
         .ToList();
```

---

## 7) Paging with `Take` and `Skip`

Useful for “Top 10 newest” or pagination.

```csharp
var newest3 =
    tasks.OrderByDescending(t => t.CreatedAt)
         .Take(3)
         .ToList();

var page2Size2 =
    tasks.OrderBy(t => t.Id)
         .Skip(2)   // skip first 2
         .Take(2)   // take next 2
         .ToList();
```

---

## 8) Aggregations: `Count`, `Sum`, `Average`, `Min`, `Max`

```csharp
int total = tasks.Count();
int done = tasks.Count(t => t.Completed);
int pending = tasks.Count(t => !t.Completed);

int maxId = tasks.Max(t => t.Id);
int minId = tasks.Min(t => t.Id);

// Example: average title length
double avgTitleLength = tasks.Average(t => t.Title.Length);
```

---

## 9) Existence checks: `Any` and `All`

```csharp
bool anyDone = tasks.Any(t => t.Completed);
bool allDone = tasks.All(t => t.Completed);
bool anyLab = tasks.Any(t => t.Title.Contains("Lab", StringComparison.OrdinalIgnoreCase));
```

These are perfect for validation in GUI logic:

- “Do we have any tasks loaded?”
- “Are all tasks completed?”

---

## 10) Picking one item: `First`, `FirstOrDefault`, `Single`

### `First`

- returns the first match
- throws exception if none found

```csharp
var firstDone = tasks.First(t => t.Completed);
```

### `FirstOrDefault` (safer)

- returns first match or `null` (for reference types)

```csharp
var maybeTask = tasks.FirstOrDefault(t => t.Id == 999);
if (maybeTask == null)
{
    Console.WriteLine("Not found.");
}
```

### `Single` (very strict)

- means: _there must be exactly one match_
- throws if 0 matches OR more than 1 match

```csharp
var exactlyOne = tasks.Single(t => t.Id == 3);
```

**Rule:** In apps, default to `FirstOrDefault` unless you truly need strictness.

---

## 11) Grouping with `GroupBy`

### Tasks per user

```csharp
var grouped =
    tasks.GroupBy(t => t.UserId)
         .Select(g => new
         {
             UserId = g.Key,
             Total = g.Count(),
             Done = g.Count(x => x.Completed)
         })
         .OrderBy(x => x.UserId)
         .ToList();
```

This is how you generate dashboard-like summaries.

---

## 12) Deferred vs Immediate execution (crucial concept)

LINQ queries are often **deferred**: they don’t run until you iterate them.

```csharp
var query = tasks.Where(t => !t.Completed); // not executed yet

tasks.Add(new TaskItem { Id = 6, Title = "New task", Completed = false, UserId = 1, CreatedAt = DateTime.Now });

// Now executed (includes the new task)
foreach (var t in query)
    Console.WriteLine(t.Title);
```

### Force immediate execution:

```csharp
var listNow = tasks.Where(t => !t.Completed).ToList(); // snapshot NOW
```

**GUI tip:** If you’re binding results to a grid and don’t want them changing unexpectedly, use `ToList()`.

---

## 13) Real WPF scenario: Search box filtering (what you’ll do constantly)

Assume you loaded tasks from API or SQLite into `_allTasks`.

```csharp
// Example: called from SearchBox_TextChanged
string q = SearchBox.Text.Trim();

var filtered = string.IsNullOrWhiteSpace(q)
    ? _allTasks
    : _allTasks.Where(t => t.Title.Contains(q, StringComparison.OrdinalIgnoreCase)).ToList();

TasksGrid.ItemsSource = filtered;
```

This is exactly how Lab 3 filtering works, but now you understand the LINQ behind it.

---

## 14) Common LINQ mistakes (avoid these)

1. **Forgetting `.ToList()` when needed**
   - You bind to a query, later modify the source list, UI results “change mysteriously”.

2. **Using `Single` when you really want `FirstOrDefault`**
   - Causes crashes when data isn’t perfect.

3. **Multiple enumeration**
   - Running `.Where(...).Count()` and then `.Where(...).ToList()` repeats work.
   - Save it:

```csharp
var filtered = tasks.Where(t => !t.Completed).ToList();
int count = filtered.Count;
```

4. **Null strings**
   - If `Title` might be null, protect it:

```csharp
.Where(t => (t.Title ?? "").Contains(q, StringComparison.OrdinalIgnoreCase))
```

---

## Practice (in class / homework)

Given a `List<TaskItem> tasks`:

1. Show all incomplete tasks sorted by newest first
2. Show the top 5 longest titles (by length)
3. For each user, show: total tasks, completed tasks
4. Check if any task contains “lab” (case-insensitive)
