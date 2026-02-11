# Lecture 2 — File System API (C# / .NET)

**Course:** Application Development 1 (Desktop)  
**Goal today:** Read/write files, manage folders, build correct paths, and avoid the classic beginner mistakes (hard-coded paths, crashes on bad input, locked files).

---

## Learning outcomes

By the end of this lecture, you can:

- Build safe file paths with `Path`
- Create/read/write files using `File`
- Create/list folders using `Directory`
- Copy/move/rename/delete files safely
- Read/write text using streams (`StreamReader`, `StreamWriter`)
- Use `FileInfo` / `DirectoryInfo` for metadata (size, dates)
- Handle common exceptions and file-lock issues
- Save app data to user folders (Documents / AppData)

---

## 0) Namespaces you’ll use

```csharp
using System;
using System.IO;
using System.Text;
using System.Text.Json;
```

`System.IO` is the main file system namespace.

---

## 1) Paths: the #1 thing students mess up

### Absolute vs Relative

- **Absolute path:** `C:\Users\adinashby\Documents\data.txt`
- **Relative path:** `data.txt` (relative to the _current working directory_)

**Strong rule:** _Never hard-code file paths._ Always build them.

### `Path.Combine` (use this always)

```csharp
string folder = "Data";
string fileName = "notes.txt";

string fullPath = Path.Combine(folder, fileName);
Console.WriteLine(fullPath); // Data\notes.txt on Windows
```

### Current directory (what your app thinks “here” is)

```csharp
Console.WriteLine(Directory.GetCurrentDirectory());
```

This can change depending on how you run the app. For predictable paths, use one of these:

### App base directory (where the app is running from)

```csharp
string baseDir = AppContext.BaseDirectory;
Console.WriteLine(baseDir);
```

### User folders (best for desktop apps)

**Best practice:** store user data in `Documents` or `AppData`, not inside your project folder.

```csharp
string docs = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
string appFolder = Path.Combine(docs, "MyDesktopApp");
Directory.CreateDirectory(appFolder);

string settingsPath = Path.Combine(appFolder, "settings.txt");
Console.WriteLine(settingsPath);
```

---

## 2) File basics with `File` (fast and easy)

### Write text

```csharp
string path = "hello.txt";
File.WriteAllText(path, "Hello file system!");
```

### Read text

```csharp
string content = File.ReadAllText("hello.txt");
Console.WriteLine(content);
```

### Append text

```csharp
File.AppendAllText("hello.txt", "\nAnother line.");
```

### Read/write lines

```csharp
string[] lines = { "Line 1", "Line 2", "Line 3" };
File.WriteAllLines("lines.txt", lines);

var readLines = File.ReadAllLines("lines.txt");
foreach (var line in readLines)
    Console.WriteLine(line);
```

### Check existence before reading (avoid crashes)

```csharp
string path = "missing.txt";

if (File.Exists(path))
{
    Console.WriteLine(File.ReadAllText(path));
}
else
{
    Console.WriteLine("File not found.");
}
```

---

## 3) Directory basics with `Directory`

### Create folder

```csharp
Directory.CreateDirectory("Data"); // safe even if it exists
```

### List files

```csharp
Directory.CreateDirectory("Data");
File.WriteAllText(Path.Combine("Data", "a.txt"), "A");
File.WriteAllText(Path.Combine("Data", "b.txt"), "B");

string[] files = Directory.GetFiles("Data");
foreach (var f in files)
    Console.WriteLine(f);
```

### List files with a pattern

```csharp
var txtFiles = Directory.GetFiles("Data", "*.txt");
foreach (var f in txtFiles)
    Console.WriteLine(f);
```

### Recursive search (subfolders too)

```csharp
var allTxt = Directory.GetFiles("Data", "*.txt", SearchOption.AllDirectories);
foreach (var f in allTxt)
    Console.WriteLine(f);
```

**Tip:** Prefer `EnumerateFiles` when listing huge folders (streams results instead of loading everything).

```csharp
foreach (var f in Directory.EnumerateFiles("Data", "*.txt", SearchOption.AllDirectories))
{
    Console.WriteLine(f);
}
```

---

## 4) Copy / Move / Rename / Delete

### Copy

```csharp
File.Copy("hello.txt", "hello_copy.txt", overwrite: true);
```

### Move (also works as rename if same folder)

```csharp
File.Move("hello_copy.txt", "renamed.txt", overwrite: true);
```

### Delete (be careful)

```csharp
if (File.Exists("renamed.txt"))
    File.Delete("renamed.txt");
```

### Delete a folder

```csharp
if (Directory.Exists("OldData"))
    Directory.Delete("OldData", recursive: true);
```

---

## 5) Metadata with `FileInfo` / `DirectoryInfo`

`File`/`Directory` are quick “static helpers”.
`FileInfo`/`DirectoryInfo` are nicer when you want **properties**.

```csharp
var info = new FileInfo("hello.txt");

Console.WriteLine(info.FullName);
Console.WriteLine(info.Length); // bytes
Console.WriteLine(info.CreationTime);
Console.WriteLine(info.LastWriteTime);
```

---

## 6) Streams (when files get big or you need control)

### Why streams?

`File.ReadAllText` loads the whole file into memory.
For big files, read line-by-line.

### Writing with `StreamWriter` (`using` auto-disposes)

```csharp
using var writer = new StreamWriter("log.txt", append: true);
writer.WriteLine($"[{DateTime.Now}] App started");
writer.WriteLine("Another log line");
```

### Reading with `StreamReader`

```csharp
using var reader = new StreamReader("log.txt");
string? line;

while ((line = reader.ReadLine()) != null)
{
    Console.WriteLine(line);
}
```

### Encoding (when accents/Unicode matter)

Most of the time UTF-8 is what you want.

```csharp
File.WriteAllText("utf8.txt", "Montréal — café", Encoding.UTF8);
string text = File.ReadAllText("utf8.txt", Encoding.UTF8);
Console.WriteLine(text);
```

---

## 7) Exceptions you must learn (or your app will feel “buggy”)

Common exceptions:

- `FileNotFoundException`
- `DirectoryNotFoundException`
- `UnauthorizedAccessException` (no permission)
- `IOException` (file locked, disk issues, etc.)

### Basic try/catch pattern (use this in real apps)

```csharp
try
{
    string content = File.ReadAllText("data.txt");
    Console.WriteLine(content);
}
catch (FileNotFoundException)
{
    Console.WriteLine("The file does not exist.");
}
catch (UnauthorizedAccessException)
{
    Console.WriteLine("No permission to access this file.");
}
catch (IOException ex)
{
    Console.WriteLine($"I/O error: {ex.Message}");
}
```

**Strong rule:** don’t let your app crash on file operations. Handle failures gracefully.

---

## 8) Async file I/O (keeps desktop apps responsive)

In desktop apps, long file reads/writes can freeze the UI.
Async prevents that.

```csharp
string path = "big.txt";

await File.WriteAllTextAsync(path, "Some content...");
string text = await File.ReadAllTextAsync(path);

Console.WriteLine(text);
```

---

## 9) Mini Demo: “Save Settings” to a safe user folder

Copy/paste into `Program.cs`:

```csharp
using System;
using System.IO;
using System.Text.Json;

var docs = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
var appFolder = Path.Combine(docs, "MyDesktopApp");
Directory.CreateDirectory(appFolder);

var settingsPath = Path.Combine(appFolder, "settings.json");

var settings = new AppSettings
{
    Theme = "Dark",
    FontSize = 16,
    LastOpenedProject = "Lecture2"
};

// Save
string json = JsonSerializer.Serialize(settings, new JsonSerializerOptions { WriteIndented = true });
File.WriteAllText(settingsPath, json);

// Load
string loadedJson = File.ReadAllText(settingsPath);
var loaded = JsonSerializer.Deserialize<AppSettings>(loadedJson);

Console.WriteLine("Saved to:");
Console.WriteLine(settingsPath);

Console.WriteLine("\nLoaded settings:");
Console.WriteLine($"Theme: {loaded?.Theme}");
Console.WriteLine($"FontSize: {loaded?.FontSize}");
Console.WriteLine($"LastOpenedProject: {loaded?.LastOpenedProject}");

class AppSettings
{
    public string Theme { get; set; } = "Light";
    public int FontSize { get; set; } = 14;
    public string LastOpenedProject { get; set; } = "";
}
```

---

## 10) Best practices (do these and you’ll look like a pro)

- **Always** build paths with `Path.Combine` (never hardcode slashes)
- Store user data in **Documents/AppData**, not the project folder
- Check with `File.Exists` / `Directory.Exists` before reading
- Use `try/catch` around file operations
- Use streams for large files or line-by-line processing
- In desktop apps, prefer **async** file I/O for anything non-trivial

---

## Practice exercises

1. **Notes app (console):**
   Ask the user for a note, append it to `notes.txt` in a folder named `Data`.

2. **File viewer:**
   Ask for a file path, if it exists print its contents, otherwise show a friendly message.

3. **Directory report:**
   Ask for a folder path, list all `.txt` files recursively, and print each file name + size in bytes.

4. **Settings file:**
   Create `settings.json` in Documents, write a class to it, then load it back and print values.
