# Lecture 7 — SQLite Database (WPF + C# CRUD)

**Course:** Application Development 1 (Desktop)  
**Goal today:** Build a WPF desktop app that stores data locally using **SQLite** and supports full **CRUD** (Create, Read, Update, Delete).

---

## Learning outcomes

By the end of this lecture, you can:

- Explain what SQLite is and when to use it
- Add the SQLite NuGet package to a .NET project
- Create a database file and tables automatically
- Perform CRUD with parameterized SQL (no SQL injection)
- Display data in a WPF `DataGrid`
- Wire up Add / Update / Delete / Refresh buttons
- Understand the idea of repository/service classes (clean structure)

---

## 1) What is SQLite?

SQLite is a lightweight **file-based** database:

- the database is a single `.db` file
- no server installation
- perfect for desktop apps (local storage, offline apps)

---

## 2) Create a WPF project + install SQLite package

Create project:

```bash
dotnet new wpf -n Lecture7Sqlite
cd Lecture7Sqlite
```

Install package:

```bash
dotnet add package Microsoft.Data.Sqlite
```

`Microsoft.Data.Sqlite` is Microsoft’s lightweight SQLite provider for .NET.

---

## 3) App we’re building: “Tasks” (local CRUD)

We’ll store tasks in a table:

**Table:** `Tasks`

- `Id` (INTEGER PRIMARY KEY AUTOINCREMENT)
- `Title` (TEXT NOT NULL)
- `Completed` (INTEGER 0/1)
- `CreatedAt` (TEXT ISO string)

---

## 4) WPF UI (MainWindow.xaml)

Replace `MainWindow.xaml` with this:

```xml
<Window x:Class="Lecture7Sqlite.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Lecture 7 - SQLite CRUD" Height="520" Width="900">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <TextBlock Text="SQLite Tasks (CRUD)" FontSize="18" Margin="0,0,0,10"/>

        <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,0,0,10">
            <TextBox x:Name="TitleBox"
                     Width="380"
                     Height="30"
                     VerticalContentAlignment="Center"
                     ToolTip="Task title"/>

            <CheckBox x:Name="CompletedBox"
                      Content="Completed"
                      Margin="10,0,0,0"
                      VerticalAlignment="Center"/>

            <Button Content="Add" Width="90" Height="30" Margin="10,0,0,0" Click="Add_Click"/>
            <Button Content="Update" Width="90" Height="30" Margin="10,0,0,0" Click="Update_Click"/>
            <Button Content="Delete" Width="90" Height="30" Margin="10,0,0,0" Click="Delete_Click"/>
            <Button Content="Refresh" Width="90" Height="30" Margin="10,0,0,0" Click="Refresh_Click"/>
        </StackPanel>

        <DataGrid x:Name="TasksGrid"
                  Grid.Row="2"
                  AutoGenerateColumns="False"
                  CanUserAddRows="False"
                  IsReadOnly="True"
                  SelectionChanged="TasksGrid_SelectionChanged">
            <DataGrid.Columns>
                <DataGridTextColumn Header="Id" Binding="{Binding Id}" Width="80"/>
                <DataGridTextColumn Header="Title" Binding="{Binding Title}" Width="*"/>
                <DataGridCheckBoxColumn Header="Completed" Binding="{Binding Completed}" Width="110"/>
                <DataGridTextColumn Header="Created At" Binding="{Binding CreatedAt}" Width="190"/>
            </DataGrid.Columns>
        </DataGrid>
    </Grid>
</Window>
```

---

## 5) Model: `TaskItem.cs`

Create `TaskItem.cs`:

```csharp
namespace Lecture7Sqlite;

public class TaskItem
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public bool Completed { get; set; }
    public string CreatedAt { get; set; } = "";
}
```

---

## 6) Database service: `SqliteTaskService.cs`

Create `SqliteTaskService.cs`:

```csharp
using Microsoft.Data.Sqlite;
using System;
using System.Collections.Generic;
using System.IO;

namespace Lecture7Sqlite;

public class SqliteTaskService
{
    private readonly string _dbPath;
    private readonly string _connectionString;

    public SqliteTaskService()
    {
        // Store in Documents so it persists and is easy to find
        string docs = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
        string folder = Path.Combine(docs, "Lecture7Sqlite");
        Directory.CreateDirectory(folder);

        _dbPath = Path.Combine(folder, "tasks.db");
        _connectionString = $"Data Source={_dbPath}";
    }

    public void Initialize()
    {
        using var conn = new SqliteConnection(_connectionString);
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText =
        @"
        CREATE TABLE IF NOT EXISTS Tasks (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            Title TEXT NOT NULL,
            Completed INTEGER NOT NULL,
            CreatedAt TEXT NOT NULL
        );
        ";
        cmd.ExecuteNonQuery();
    }

    public List<TaskItem> GetAll()
    {
        var list = new List<TaskItem>();

        using var conn = new SqliteConnection(_connectionString);
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText =
        @"
        SELECT Id, Title, Completed, CreatedAt
        FROM Tasks
        ORDER BY Id DESC;
        ";

        using var reader = cmd.ExecuteReader();
        while (reader.Read())
        {
            list.Add(new TaskItem
            {
                Id = reader.GetInt32(0),
                Title = reader.GetString(1),
                Completed = reader.GetInt32(2) == 1,
                CreatedAt = reader.GetString(3)
            });
        }

        return list;
    }

    public int Add(string title, bool completed)
    {
        using var conn = new SqliteConnection(_connectionString);
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText =
        @"
        INSERT INTO Tasks (Title, Completed, CreatedAt)
        VALUES ($title, $completed, $createdAt);
        SELECT last_insert_rowid();
        ";

        cmd.Parameters.AddWithValue("$title", title);
        cmd.Parameters.AddWithValue("$completed", completed ? 1 : 0);
        cmd.Parameters.AddWithValue("$createdAt", DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"));

        // last_insert_rowid() returns the new row's ID
        long newId = (long)cmd.ExecuteScalar();
        return (int)newId;
    }

    public void Update(int id, string title, bool completed)
    {
        using var conn = new SqliteConnection(_connectionString);
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText =
        @"
        UPDATE Tasks
        SET Title = $title,
            Completed = $completed
        WHERE Id = $id;
        ";

        cmd.Parameters.AddWithValue("$id", id);
        cmd.Parameters.AddWithValue("$title", title);
        cmd.Parameters.AddWithValue("$completed", completed ? 1 : 0);

        cmd.ExecuteNonQuery();
    }

    public void Delete(int id)
    {
        using var conn = new SqliteConnection(_connectionString);
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText =
        @"
        DELETE FROM Tasks
        WHERE Id = $id;
        ";

        cmd.Parameters.AddWithValue("$id", id);
        cmd.ExecuteNonQuery();
    }
}
```

**Why parameters matter:**
They prevent SQL injection and handle quotes safely.

---

## 7) Code-behind: `MainWindow.xaml.cs`

Replace `MainWindow.xaml.cs` with:

```csharp
using System.Collections.Generic;
using System.Windows;

namespace Lecture7Sqlite;

public partial class MainWindow : Window
{
    private readonly SqliteTaskService _db = new SqliteTaskService();
    private int? _selectedId = null;

    public MainWindow()
    {
        InitializeComponent();

        _db.Initialize();
        RefreshGrid();
    }

    private void RefreshGrid()
    {
        List<TaskItem> tasks = _db.GetAll();
        TasksGrid.ItemsSource = tasks;
    }

    private void Add_Click(object sender, RoutedEventArgs e)
    {
        string title = TitleBox.Text.Trim();
        bool completed = CompletedBox.IsChecked == true;

        if (string.IsNullOrWhiteSpace(title))
        {
            MessageBox.Show("Please enter a title.");
            return;
        }

        _db.Add(title, completed);

        TitleBox.Clear();
        CompletedBox.IsChecked = false;
        _selectedId = null;

        RefreshGrid();
    }

    private void Update_Click(object sender, RoutedEventArgs e)
    {
        if (_selectedId == null)
        {
            MessageBox.Show("Select a row first.");
            return;
        }

        string title = TitleBox.Text.Trim();
        bool completed = CompletedBox.IsChecked == true;

        if (string.IsNullOrWhiteSpace(title))
        {
            MessageBox.Show("Please enter a title.");
            return;
        }

        _db.Update(_selectedId.Value, title, completed);

        TitleBox.Clear();
        CompletedBox.IsChecked = false;
        _selectedId = null;

        RefreshGrid();
    }

    private void Delete_Click(object sender, RoutedEventArgs e)
    {
        if (_selectedId == null)
        {
            MessageBox.Show("Select a row first.");
            return;
        }

        _db.Delete(_selectedId.Value);

        TitleBox.Clear();
        CompletedBox.IsChecked = false;
        _selectedId = null;

        RefreshGrid();
    }

    private void Refresh_Click(object sender, RoutedEventArgs e)
    {
        RefreshGrid();
    }

    private void TasksGrid_SelectionChanged(object sender, System.Windows.Controls.SelectionChangedEventArgs e)
    {
        if (TasksGrid.SelectedItem is not TaskItem selected)
            return;

        _selectedId = selected.Id;
        TitleBox.Text = selected.Title;
        CompletedBox.IsChecked = selected.Completed;
    }
}
```

---

## 8) What students must remember

- SQLite is a local `.db` file (portable, fast, offline)
- Always create tables with `CREATE TABLE IF NOT EXISTS`
- Always use parameterized SQL (`$title`) instead of string concatenation
- Your WPF UI should:
  - show data in `DataGrid`
  - use selection to load into input fields
  - perform CRUD through a service class

---

## Practice extensions (optional)

1. Add a **Search** TextBox to filter tasks by title (like Lab 3)
2. Add a **Delete All Completed** button:

   ```sql
   DELETE FROM Tasks WHERE Completed = 1;
   ```

3. Add a `DueDate` column to the table and update UI
4. Make all DB calls async using `Task.Run` (to practice threading patterns)
