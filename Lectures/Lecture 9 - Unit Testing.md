# Lecture 9 — Unit Testing in C# (xUnit)

**Course:** Application Development 1 (Desktop)  
**Goal today:** Write automated tests for your C# code so you can change/refactor safely without breaking features.

---

## Learning outcomes

By the end of this lecture, you can:

- Explain what a **unit test** is (and what it isn’t)
- Create a test project and run tests with `dotnet test`
- Write tests using the **Arrange–Act–Assert** pattern
- Test normal cases, edge cases, and exceptions
- Write **data-driven tests** (`[Theory]`, `[InlineData]`)
- Test **async** methods
- Understand when to **mock** dependencies (Web API / time / randomness)
- Separate WPF UI from testable logic

---

## 1) What is a Unit Test?

A **unit test** verifies one small piece of logic in isolation:

- a method
- a class
- a rule (validation)

**Unit tests should be:**

- fast (milliseconds)
- deterministic (same result every run)
- isolated (no dependency on UI, internet, real database files)

**Not unit tests:**

- clicking buttons in WPF
- calling real Web APIs
- using your real production database file  
  Those are **integration tests** (still useful, but different).

---

## 2) Which test framework?

Common C# test frameworks:

- **xUnit** (very common in modern .NET)
- NUnit
- MSTest

We’ll use **xUnit** because it works great with `dotnet test`.

---

## 3) Recommended project structure (so WPF code is testable)

✅ Put your “logic” in a **class library**.  
WPF should call that library.

Example:

- `AppDev1.Core` (class library) → pure logic/services/models
- `AppDev1.Wpf` (WPF UI) → XAML + event handlers
- `AppDev1.Tests` (xUnit tests) → tests for Core

---

## 4) Create projects (CLI)

From an empty folder:

```bash
dotnet new sln -n AppDev1

dotnet new classlib -n AppDev1.Core
dotnet new wpf -n AppDev1.Wpf
dotnet new xunit -n AppDev1.Tests

dotnet sln AppDev1.sln add AppDev1.Core/AppDev1.Core.csproj
dotnet sln AppDev1.sln add AppDev1.Wpf/AppDev1.Wpf.csproj
dotnet sln AppDev1.sln add AppDev1.Tests/AppDev1.Tests.csproj

dotnet add AppDev1.Wpf/AppDev1.Wpf.csproj reference AppDev1.Core/AppDev1.Core.csproj
dotnet add AppDev1.Tests/AppDev1.Tests.csproj reference AppDev1.Core/AppDev1.Core.csproj
```

Run tests:

```bash
dotnet test
```

---

## 5) Anatomy of a test (AAA pattern)

**Arrange**: set up data
**Act**: call the method
**Assert**: verify result

---

## 6) Example 1 — testing simple methods (Core)

Create `AppDev1.Core/MathUtils.cs`:

```csharp
namespace AppDev1.Core;

public static class MathUtils
{
    public static int Add(int x, int y) => x + y;

    public static bool IsEven(int n) => n % 2 == 0;
}
```

Create `AppDev1.Tests/MathUtilsTests.cs`:

```csharp
using AppDev1.Core;
using Xunit;

public class MathUtilsTests
{
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsCorrectSum()
    {
        // Arrange
        int a = 3, b = 7;

        // Act
        int result = MathUtils.Add(a, b);

        // Assert
        Assert.Equal(10, result);
    }

    [Theory]
    [InlineData(2, true)]
    [InlineData(3, false)]
    [InlineData(0, true)]
    [InlineData(-4, true)]
    public void IsEven_VariousInputs_ReturnsExpected(int n, bool expected)
    {
        // Act
        bool result = MathUtils.IsEven(n);

        // Assert
        Assert.Equal(expected, result);
    }
}
```

---

## 7) Assertions you’ll use constantly

```csharp
Assert.True(condition);
Assert.False(condition);
Assert.Equal(expected, actual);
Assert.NotNull(value);
Assert.Null(value);
Assert.Contains("abc", someString);
```

---

## 8) Testing exceptions (input validation)

Core method:

```csharp
namespace AppDev1.Core;

public static class GradeUtils
{
    public static string ToLetter(int grade)
    {
        if (grade < 0 || grade > 100)
            throw new ArgumentOutOfRangeException(nameof(grade));

        if (grade >= 90) return "A";
        if (grade >= 80) return "B";
        if (grade >= 70) return "C";
        if (grade >= 60) return "D";
        return "F";
    }
}
```

Test:

```csharp
using AppDev1.Core;
using Xunit;

public class GradeUtilsTests
{
    [Theory]
    [InlineData(90, "A")]
    [InlineData(80, "B")]
    [InlineData(70, "C")]
    [InlineData(60, "D")]
    [InlineData(59, "F")]
    public void ToLetter_ValidGrades_ReturnsExpected(int grade, string expected)
    {
        Assert.Equal(expected, GradeUtils.ToLetter(grade));
    }

    [Theory]
    [InlineData(-1)]
    [InlineData(101)]
    public void ToLetter_InvalidGrades_Throws(int grade)
    {
        Assert.Throws<ArgumentOutOfRangeException>(() => GradeUtils.ToLetter(grade));
    }
}
```

---

## 9) Example 2 — Testing your Magic Square validator (from Lab 2)

Put the validator into Core: `AppDev1.Core/MagicSquare.cs`

```csharp
namespace AppDev1.Core;

public static class MagicSquare
{
    public static bool IsMagicSquare(int[,] matrix)
    {
        int n = matrix.GetLength(0);
        if (n == 0 || n != matrix.GetLength(1)) return false;

        int max = n * n;
        bool[] seen = new bool[max + 1];

        for (int r = 0; r < n; r++)
        for (int c = 0; c < n; c++)
        {
            int v = matrix[r, c];
            if (v < 1 || v > max) return false;
            if (seen[v]) return false;
            seen[v] = true;
        }

        int target = 0;
        for (int c = 0; c < n; c++) target += matrix[0, c];

        for (int r = 0; r < n; r++)
        {
            int sum = 0;
            for (int c = 0; c < n; c++) sum += matrix[r, c];
            if (sum != target) return false;
        }

        for (int c = 0; c < n; c++)
        {
            int sum = 0;
            for (int r = 0; r < n; r++) sum += matrix[r, c];
            if (sum != target) return false;
        }

        int d1 = 0, d2 = 0;
        for (int i = 0; i < n; i++)
        {
            d1 += matrix[i, i];
            d2 += matrix[i, n - 1 - i];
        }

        return d1 == target && d2 == target;
    }
}
```

Tests: `AppDev1.Tests/MagicSquareTests.cs`

```csharp
using AppDev1.Core;
using Xunit;

public class MagicSquareTests
{
    [Fact]
    public void IsMagicSquare_Valid3x3_ReturnsTrue()
    {
        int[,] m =
        {
            { 4, 9, 2 },
            { 3, 5, 7 },
            { 8, 1, 6 }
        };

        Assert.True(MagicSquare.IsMagicSquare(m));
    }

    [Fact]
    public void IsMagicSquare_NotMagic_ReturnsFalse()
    {
        int[,] m =
        {
            { 1, 2, 3 },
            { 4, 5, 6 },
            { 7, 8, 9 }
        };

        Assert.False(MagicSquare.IsMagicSquare(m));
    }

    [Fact]
    public void IsMagicSquare_DuplicateNumbers_ReturnsFalse()
    {
        int[,] m =
        {
            { 4, 9, 2 },
            { 3, 5, 7 },
            { 8, 1, 4 } // duplicate 4
        };

        Assert.False(MagicSquare.IsMagicSquare(m));
    }
}
```

---

## 10) Testing async methods

If a method returns `Task`, your test should also be `async Task`.

Example Core method:

```csharp
namespace AppDev1.Core;

public static class DelayDemo
{
    public static async Task<int> GetValueAsync()
    {
        await Task.Delay(50);
        return 42;
    }
}
```

Test:

```csharp
using AppDev1.Core;
using Xunit;

public class DelayDemoTests
{
    [Fact]
    public async Task GetValueAsync_Returns42()
    {
        int value = await DelayDemo.GetValueAsync();
        Assert.Equal(42, value);
    }
}
```

Testing async exceptions:

```csharp
await Assert.ThrowsAsync<InvalidOperationException>(async () =>
{
    await SomeAsyncMethodThatThrows();
});
```

---

## 11) Mocking (when your code depends on Web APIs)

If you test Web API code by calling the real internet, tests become:

- slow
- flaky
- dependent on network/server

Better: mock `HttpClient` using a fake `HttpMessageHandler`.

Example (minimal pattern):

```csharp
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public class FakeHttpHandler : HttpMessageHandler
{
    private readonly string _responseJson;

    public FakeHttpHandler(string responseJson) => _responseJson = responseJson;

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        var resp = new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(_responseJson)
        };
        return Task.FromResult(resp);
    }
}
```

Then create `HttpClient` with it in your tests:

```csharp
var handler = new FakeHttpHandler("[{ \"id\": 1, \"title\": \"Test\", \"completed\": false, \"userId\": 1 }]");
var http = new HttpClient(handler);
```

**Key idea:** you control the JSON → your test is deterministic.

---

## 12) What to test in WPF apps

✅ Test these:

- validation methods (TryParse-like logic wrapped into methods)
- services (SQLite CRUD logic, API parsing logic)
- business rules (grading, magic square rules, filtering)

❌ Don’t unit test:

- Button layout
- XAML appearance
- “Does the window open?” (that’s UI/system testing)

**Rule:** keep UI “thin” and logic “thick” in Core/services.

---

## 13) Good testing habits

- Test names should describe behavior:
  - `Method_Input_ExpectedResult`

- Test one behavior at a time
- Prefer small, pure functions
- Avoid order dependency (tests must pass in any order)
- Avoid real network calls
- Avoid writing to permanent files; use temp files if needed

---

## Practice (Lab-style)

1. Write tests for your `GenerateMagicSquare(n)`:
   - `n` odd generates a valid magic square (use `IsMagicSquare`)

2. Add a `FilterByTitle(List<TaskItem>, string query)` method and test:
   - empty query returns all
   - query matches case-insensitively

3. Add `GradeUtils.ToLetter` tests (boundary values: 59/60, 69/70, etc.)
