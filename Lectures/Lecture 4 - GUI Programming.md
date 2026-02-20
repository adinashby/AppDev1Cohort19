# Lecture 4 — GUI Programming with WPF (C#)

**Course:** Application Development 1 (Desktop)  
**Goal today:** Build your first WPF app, understand the event-driven model, layout basics, and safe async patterns so your UI never freezes.

---

## Learning outcomes

By the end of this lecture, you can:

- Create a WPF project and understand its files
- Explain the roles of **XAML** and **code-behind**
- Use common controls: `TextBox`, `Button`, `Label/TextBlock`, `ListBox`
- Use layout containers: `Grid`, `StackPanel`
- Handle events like `Click`
- Validate input and update UI safely
- Use `async/await` in button handlers to keep the UI responsive
- Understand the idea of **data binding** (preview for later lectures)

---

## 0) Create a WPF project (VS Code / CLI)

> WPF runs on Windows. Use Windows machine/VM.

```bash
dotnet new wpf -n Lecture4Wpf
cd Lecture4Wpf
dotnet run
```

If you’re using Visual Studio, create: **WPF App (.NET)**.

---

## 1) WPF project structure (must know)

Typical files:

- `App.xaml`
  Application-wide resources and startup settings.

- `MainWindow.xaml`
  The UI layout written in XAML.

- `MainWindow.xaml.cs`
  Code-behind: event handlers + logic.

**XAML = markup describing UI**
**C# code-behind = behavior**

---

## 2) Your first WPF window (Hello UI)

### `MainWindow.xaml`

```xml
<Window x:Class="Lecture4Wpf.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Lecture 4 - WPF" Height="250" Width="420">
    <Grid Margin="16">
        <StackPanel>
            <TextBlock Text="Welcome to WPF" FontSize="22" Margin="0,0,0,10"/>
            <Button Content="Click Me" Width="120" Height="35"/>
        </StackPanel>
    </Grid>
</Window>
```

Run it and you should see a window with a title, a text header, and a button.

---

## 3) Layout fundamentals (the part that makes or breaks your UI)

### Grid vs StackPanel

- **StackPanel:** stacks items vertically or horizontally. Simple.
- **Grid:** real app layout. Rows and columns.

#### Grid example (2 rows)

```xml
<Grid Margin="16">
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
    </Grid.RowDefinitions>

    <TextBlock Text="Top area" FontSize="18"/>
    <TextBlock Grid.Row="1" Text="Main area" VerticalAlignment="Center" HorizontalAlignment="Center"/>
</Grid>
```

**Important:**

- `Auto` = size to content
- `*` = take remaining space

---

## 4) Controls you’ll use constantly

### Common controls

- `TextBox` → user input
- `Button` → actions
- `TextBlock` → text display
- `Label` → text + access keys (less used than TextBlock)
- `ListBox` → list of items
- `CheckBox` / `ComboBox` → choices

---

## 5) Events (WPF is event-driven)

In WPF, users don’t call your methods directly—**events** do:

- Button click triggers `Click`
- Text changes triggers `TextChanged`
- Window loads triggers `Loaded`

---

## 6) Mini app #1: Greeting app (XAML + event handler)

### `MainWindow.xaml`

```xml
<Window x:Class="Lecture4Wpf.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Greeting App" Height="260" Width="460">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <TextBlock Text="Enter your name:" FontSize="16" />
        <TextBox x:Name="NameTextBox" Grid.Row="1" Height="30" Margin="0,6,0,10"/>

        <Button Grid.Row="2" Content="Greet" Width="120" Height="35"
                Click="GreetButton_Click"/>

        <TextBlock x:Name="OutputTextBlock" Grid.Row="3" FontSize="16"
                   Margin="0,12,0,0"/>
    </Grid>
</Window>
```

### `MainWindow.xaml.cs`

```csharp
using System.Windows;

namespace Lecture4Wpf
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private void GreetButton_Click(object sender, RoutedEventArgs e)
        {
            string name = NameTextBox.Text.Trim();

            if (string.IsNullOrWhiteSpace(name))
            {
                OutputTextBlock.Text = "Please enter a name.";
                return;
            }

            OutputTextBlock.Text = $"Hello, {name}!";
        }
    }
}
```

---

## 7) Input validation: don’t let bad input crash your app

### Mini app #2: Add two numbers

#### XAML (replace window content with this)

```xml
<Grid Margin="16">
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
    </Grid.RowDefinitions>

    <TextBlock Text="Enter two integers:" FontSize="16"/>

    <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,8,0,8">
        <TextBox x:Name="ABox" Width="120" Height="30" Margin="0,0,10,0"/>
        <TextBox x:Name="BBox" Width="120" Height="30"/>
    </StackPanel>

    <Button Grid.Row="2" Content="Add" Width="120" Height="35"
            Click="AddButton_Click"/>

    <TextBlock x:Name="ResultTextBlock" Grid.Row="3" FontSize="16" Margin="0,12,0,0"/>
</Grid>
```

#### Code-behind

```csharp
private void AddButton_Click(object sender, RoutedEventArgs e)
{
    if (!int.TryParse(ABox.Text, out int a) || !int.TryParse(BBox.Text, out int b))
    {
        ResultTextBlock.Text = "Invalid input. Please enter integers.";
        return;
    }

    ResultTextBlock.Text = $"{a} + {b} = {a + b}";
}
```

---

## 8) ListBox demo: a tiny “Notes” UI

### XAML snippet

```xml
<Grid Margin="16">
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
    </Grid.RowDefinitions>

    <TextBlock Text="Notes" FontSize="18" />

    <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,8,0,8">
        <TextBox x:Name="NoteBox" Width="260" Height="30" Margin="0,0,10,0"/>
        <Button Content="Add" Width="80" Height="30" Click="AddNote_Click"/>
        <Button Content="Remove" Width="80" Height="30" Margin="10,0,0,0" Click="RemoveNote_Click"/>
    </StackPanel>

    <ListBox x:Name="NotesList" Grid.Row="2"/>
</Grid>
```

### Code-behind

```csharp
private void AddNote_Click(object sender, RoutedEventArgs e)
{
    string note = NoteBox.Text.Trim();
    if (string.IsNullOrWhiteSpace(note))
        return;

    NotesList.Items.Add(note);
    NoteBox.Clear();
    NoteBox.Focus();
}

private void RemoveNote_Click(object sender, RoutedEventArgs e)
{
    if (NotesList.SelectedItem != null)
        NotesList.Items.Remove(NotesList.SelectedItem);
}
```

---

## 9) The most important GUI topic: async so your UI never freezes

### Bad (freezes UI)

```csharp
private void BadButton_Click(object sender, RoutedEventArgs e)
{
    Thread.Sleep(3000); // UI freezes
}
```

### Good (responsive UI)

```csharp
private async void GoodButton_Click(object sender, RoutedEventArgs e)
{
    await Task.Delay(3000); // UI remains responsive
}
```

### CPU-bound work (use Task.Run)

```csharp
private async void ComputeButton_Click(object sender, RoutedEventArgs e)
{
    ComputeButton.IsEnabled = false;
    OutputTextBlock.Text = "Working...";

    try
    {
        int result = await Task.Run(() => ExpensiveCalculation());
        OutputTextBlock.Text = $"Result: {result}";
    }
    finally
    {
        ComputeButton.IsEnabled = true;
    }
}

private int ExpensiveCalculation()
{
    // Simulate heavy CPU work
    int total = 0;
    for (int i = 0; i < 50_000_000; i++)
        total += i % 3;

    return total;
}
```

---

## 10) Preview: Data Binding (what we’ll move to soon)

Right now we are doing:

- manually setting `TextBlock.Text`
- manually adding items to `ListBox.Items`

That works, but it doesn’t scale.

**WPF’s superpower** is data binding + MVVM:

- UI binds to properties
- UI updates automatically when data changes

Example preview (don’t worry about MVVM today):

```xml
<TextBox Text="{Binding UserName}" />
```

We’ll build up to this properly.

---

## 11) Quick checklist: WPF habits that make you good fast

- Use `Grid` for real layouts; avoid nesting 10 StackPanels
- Name controls with `x:Name` when you need them in code-behind
- Validate user input always (`TryParse`, `IsNullOrWhiteSpace`)
- Never block UI thread (`Thread.Sleep` is basically banned in event handlers)
- Use `async void` **only** for event handlers
- Use `Task.Run` for CPU work; use `await` for I/O work
