# Lecture 1 — C# Basics and Syntax

**Course:** Application Development 1 (Desktop)  
**Language:** C# (.NET)  
**Goal today:** Get comfortable writing and running C# code, reading compiler errors, and understanding core syntax.

---

## Learning outcomes

By the end of this lecture, you can:

- Create and run a C# project in VS Code
- Use variables, constants, and basic data types
- Write expressions with operators
- Work with strings and user input
- Use `if / else`, `switch`, loops
- Write and call methods
- Understand the idea of classes/objects (preview)

---

## 0) Setup (VS Code + .NET)

### Install

- Install **.NET SDK (8+)**
- Install **VS Code**
- Install VS Code extension: **C# Dev Kit** (and/or **C#**)

### Create a new project

Open a terminal in VS Code:

```bash
dotnet new console -n Lecture1
cd Lecture1
dotnet run
```

You should see output in the terminal.

---

## 1) The structure of a C# program

Modern C# console apps often use **top-level statements** (no explicit `Main` shown):

```csharp
Console.WriteLine("Hello, C#!");
```

Traditional form (you’ll still see this in many codebases):

```csharp
using System;

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello, C#!");
    }
}
```

✅ In this course, we’ll usually use top-level statements to move faster, but you must recognize both.

---

## 2) Variables, constants, and basic types

### Common types

- `int` (whole number)
- `double` (decimal number)
- `bool` (true/false)
- `char` (single character)
- `string` (text)

```csharp
int age = 28;
double price = 19.99;
bool isLoggedIn = true;
char grade = 'A';
string name = "Adin";

Console.WriteLine(age);
Console.WriteLine(price);
Console.WriteLine(isLoggedIn);
Console.WriteLine(grade);
Console.WriteLine(name);
```

### `var` (type inference)

C# can infer the type from the value:

```csharp
var count = 10;       // int
var total = 10.5;     // double
var title = "Hello";  // string

Console.WriteLine(count.GetType());
Console.WriteLine(total.GetType());
Console.WriteLine(title.GetType());
```

### Constants (`const`)

A constant can’t change:

```csharp
const double TaxRate = 0.15;
// TaxRate = 0.20; // ❌ compile error
Console.WriteLine(TaxRate);
```

---

## 3) Operators (math + comparisons + logic)

### Arithmetic

```csharp
int a = 10;
int b = 3;

Console.WriteLine(a + b);  // 13
Console.WriteLine(a - b);  // 7
Console.WriteLine(a * b);  // 30
Console.WriteLine(a / b);  // 3  (integer division!)
Console.WriteLine(a % b);  // 1  (remainder)
```

### Integer vs decimal division (IMPORTANT)

```csharp
Console.WriteLine(10 / 3);       // 3
Console.WriteLine(10.0 / 3);     // 3.333...
Console.WriteLine((double)10 / 3); // 3.333...
```

### Comparisons

```csharp
int x = 5;
Console.WriteLine(x == 5);  // true
Console.WriteLine(x != 5);  // false
Console.WriteLine(x > 3);   // true
Console.WriteLine(x <= 2);  // false
```

### Logical operators

```csharp
bool hasId = true;
bool hasTicket = false;

Console.WriteLine(hasId && hasTicket); // false (AND)
Console.WriteLine(hasId || hasTicket); // true  (OR)
Console.WriteLine(!hasId);             // false (NOT)
```

---

## 4) Strings: concatenation vs interpolation

### Concatenation

```csharp
string firstName = "Ada";
string lastName = "Lovelace";

Console.WriteLine("Hello " + firstName + " " + lastName + "!");
```

### Interpolation (use this most of the time)

```csharp
Console.WriteLine($"Hello {firstName} {lastName}!");
```

### Useful string operations

```csharp
string text = "  Hello World  ";

Console.WriteLine(text.Trim());          // removes outer spaces
Console.WriteLine(text.ToUpper());       // HELLO WORLD
Console.WriteLine(text.Contains("World"));// true
Console.WriteLine(text.Replace("World", "C#")); // Hello C#
```

---

## 5) Console input + parsing

`Console.ReadLine()` returns a **string** (or possibly `null`).

```csharp
Console.Write("Enter your name: ");
string? userName = Console.ReadLine();

Console.WriteLine($"Hi, {userName}!");
```

### Parsing integers

```csharp
Console.Write("Enter your age: ");
string? input = Console.ReadLine();

int age = int.Parse(input!); // "!" = trust me it's not null
Console.WriteLine($"Next year you'll be {age + 1}.");
```

### Safer parsing (`TryParse`) — recommended

```csharp
Console.Write("Enter a number: ");
string? s = Console.ReadLine();

if (int.TryParse(s, out int number))
{
    Console.WriteLine($"You typed {number}. Square = {number * number}");
}
else
{
    Console.WriteLine("That was not a valid integer.");
}
```

#### What it does (in one sentence)

It **tries** to convert the string `s` into an `int`.

- If it succeeds → it returns `true` and puts the converted integer into `number`.
- If it fails → it returns `false` and `number` becomes `0` (default int value).

#### Why you should use it

`int.Parse(...)` **crashes** your program if the input is invalid:

```csharp
int n = int.Parse("abc"); // 💥 FormatException
```

#### `TryParse` **never throws** for bad input — it just says “nope” safely.

## 6) Control flow

### if / else

```csharp
int score = 72;

if (score >= 60)
{
    Console.WriteLine("Pass");
}
else
{
    Console.WriteLine("Fail");
}
```

### else-if chain

```csharp
int grade = 85;

if (grade >= 90) Console.WriteLine("A");
else if (grade >= 80) Console.WriteLine("B");
else if (grade >= 70) Console.WriteLine("C");
else if (grade >= 60) Console.WriteLine("D");
else Console.WriteLine("F");
```

### switch (good for fixed options)

```csharp
Console.Write("Choose a mode (easy/normal/hard): ");
string? mode = Console.ReadLine();

switch (mode)
{
    case "easy":
        Console.WriteLine("Easy mode selected.");
        break;
    case "normal":
        Console.WriteLine("Normal mode selected.");
        break;
    case "hard":
        Console.WriteLine("Hard mode selected.");
        break;
    default:
        Console.WriteLine("Unknown mode.");
        break;
}
```

---

## 7) Loops

### for loop

```csharp
for (int i = 1; i <= 5; i++)
{
    Console.WriteLine($"i = {i}");
}
```

### while loop

```csharp
int attempts = 0;

while (attempts < 3)
{
    Console.WriteLine("Try again...");
    attempts++;
}
```

### do-while loop (runs at least once)

```csharp
int n;

do
{
    Console.Write("Enter a positive number: ");
    n = int.Parse(Console.ReadLine()!);
}
while (n <= 0);

Console.WriteLine($"Accepted: {n}");
```

### foreach loop (for collections)

```csharp
string[] names = { "Ada", "Alan", "Grace" };

foreach (string n in names)
{
    Console.WriteLine(n);
}
```

---

## 8) Methods (functions)

Methods help you **reuse code** and keep programs clean.

```csharp
static int Add(int x, int y)
{
    return x + y;
}

int result = Add(3, 7);
Console.WriteLine(result);
```

### `void` methods (no return)

```csharp
static void PrintLine(string message)
{
    Console.WriteLine(message);
}

PrintLine("Hello from a method!");
```

### Method with default parameter

```csharp
static void Greet(string name = "Guest")
{
    Console.WriteLine($"Welcome, {name}!");
}

Greet();
Greet("Adin");
```

---

## 9) Arrays and Lists (quick intro)

### Arrays (fixed size)

```csharp
int[] numbers = { 10, 20, 30 };
Console.WriteLine(numbers[0]); // 10
Console.WriteLine(numbers.Length); // 3
```

### Lists (dynamic size)

Add this at the top if needed:

```csharp
using System.Collections.Generic;
```

```csharp
var items = new List<string>();
items.Add("Mouse");
items.Add("Keyboard");
items.Add("Monitor");

Console.WriteLine(items.Count);

foreach (var item in items)
{
    Console.WriteLine(item);
}
```

---

## 10) Preview: Classes and objects (just enough for today)

A **class** is a blueprint. An **object** is a real instance you create.

```csharp
class Person
{
    public string Name { get; set; }
    public int Age { get; set; }

    public void Introduce()
    {
        Console.WriteLine($"Hi, I'm {Name} and I'm {Age}.");
    }
}

var p = new Person { Name = "Ada", Age = 28 };
p.Introduce();
```

We’ll go much deeper later, especially because desktop apps are built around objects + events + UI controls.

---

## 11) Mini demo: A complete small program

Copy/paste into `Program.cs`:

```csharp
Console.WriteLine("=== Mini Demo: Simple Checkout ===");

Console.Write("Item name: ");
string item = Console.ReadLine() ?? "Unknown";

Console.Write("Price (e.g., 19.99): ");
string? priceText = Console.ReadLine();

Console.Write("Quantity: ");
string? qtyText = Console.ReadLine();

if (!double.TryParse(priceText, out double price) || !int.TryParse(qtyText, out int qty))
{
    Console.WriteLine("Invalid price or quantity.");
    return;
}

double subtotal = price * qty;
const double TaxRate = 0.14975; // example rate
double tax = subtotal * TaxRate;
double total = subtotal + tax;

Console.WriteLine();
Console.WriteLine($"Item: {item}");
Console.WriteLine($"Subtotal: {subtotal:F2}");
Console.WriteLine($"Tax: {tax:F2}");
Console.WriteLine($"Total: {total:F2}");
```

---

## Common mistakes (and how to fix them fast)

- **“; expected”** → you forgot a semicolon
- **Type mismatch** → you’re trying to put a `string` into an `int` (parse it!)
- **Index out of range** → arrays start at index 0, last index is `Length - 1`
- **Null issues** → `Console.ReadLine()` can be null; use `??` or validate input

---

## Practice exercises (do in class or as homework)

1. Ask the user for 2 integers and print:

- sum, difference, product, and (decimal) division

2. Ask the user for a grade (0–100) and output letter grade:

- A/B/C/D/F using an `if/else-if` chain

3. Print the numbers 1–50:

- Replace multiples of 3 with `"Fizz"`
- Multiples of 5 with `"Buzz"`
- Multiples of both with `"FizzBuzz"`

4. Write a method `IsEven(int n)` that returns `true/false`, then test it.
