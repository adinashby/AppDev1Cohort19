# Lecture 5 — Web API (C#) + WPF UI Demo

**Course:** Application Development 1 (Desktop)  
**Goal today:** Call a Web API from C#, parse JSON into C# objects, and display the data in a simple WPF UI without freezing the app.

**API we’ll use (example):**  
`https://jsonplaceholder.typicode.com/albums`

---

## Learning outcomes

By the end of this lecture, you can:

- Explain what a Web API is (endpoint, resource, JSON)
- Make HTTP requests in C# using `HttpClient`
- Read and interpret HTTP status codes
- Deserialize JSON into C# objects using `System.Text.Json`
- Use `async/await` correctly so the WPF UI stays responsive
- Display data in WPF with `DataGrid` (simple UI like Lecture 4)

---

## 1) What is a Web API?

A **Web API** is a server you can talk to over HTTP. You send requests like:

- **GET**: retrieve data
- **POST**: create data
- **PUT/PATCH**: update data
- **DELETE**: delete data

An **endpoint** is a URL that represents a resource, like:

- `/albums` → list of albums
- `/albums/1` → one album

This API returns **JSON**, a text format like:

```json
[
  { "userId": 1, "id": 1, "title": "quidem molestiae enim" },
  { "userId": 1, "id": 2, "title": "sunt qui excepturi placeat culpa" }
]
```

---

## 2) HTTP essentials you must know

### Status Codes (very common)

- `200 OK` → success
- `201 Created` → created successfully (often after POST)
- `400 Bad Request` → your request is wrong
- `401 Unauthorized` / `403 Forbidden` → auth/permissions issue
- `404 Not Found` → wrong endpoint / missing resource
- `500 Internal Server Error` → server crashed

### Response body

For many APIs, the “real data” is in the **response body** as JSON.

---

## 3) Calling a Web API in C# (Console demo)

This is the raw idea you’ll use in WPF too.

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        using var http = new HttpClient();

        string url = "https://jsonplaceholder.typicode.com/albums";
        HttpResponseMessage response = await http.GetAsync(url);

        Console.WriteLine($"Status: {(int)response.StatusCode} {response.StatusCode}");

        string json = await response.Content.ReadAsStringAsync();
        Console.WriteLine(json.Substring(0, Math.Min(200, json.Length)));
    }
}
```

**Key point:** `await` keeps things non-blocking. In WPF this prevents freezing.

---

## 4) JSON → C# objects (Models / DTOs)

The `/albums` JSON has fields:

- `userId` (int)
- `id` (int)
- `title` (string)

Create a model class:

```csharp
public class Album
{
    public int UserId { get; set; }
    public int Id { get; set; }
    public string Title { get; set; } = "";
}
```

Then deserialize:

```csharp
using System.Text.Json;

var options = new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true
};

List<Album>? albums = JsonSerializer.Deserialize<List<Album>>(json, options);
```

---

## 5) WPF Demo App: Load Albums and Display in a DataGrid

### Create the WPF project

```bash
dotnet new wpf -n Lecture5WebApi
cd Lecture5WebApi
dotnet run
```

---

## 6) UI (MainWindow.xaml)

This is a simple UI in the style of Lecture 4:

- Button to load
- Status text
- DataGrid to show results

Replace your `MainWindow.xaml` content with:

```xml
<Window x:Class="Lecture5WebApi.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Lecture 5 - Web API" Height="500" Width="760">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <StackPanel Orientation="Horizontal" Margin="0,0,0,10">
            <Button x:Name="LoadButton"
                    Content="Load Albums"
                    Width="130" Height="34"
                    Click="LoadButton_Click"/>

            <TextBox x:Name="SearchBox"
                     Width="260" Height="34"
                     Margin="12,0,0,0"
                     VerticalContentAlignment="Center"
                     ToolTip="Filter by title..."
                     TextChanged="SearchBox_TextChanged"/>

            <Button x:Name="ClearButton"
                    Content="Clear"
                    Width="90" Height="34"
                    Margin="12,0,0,0"
                    Click="ClearButton_Click"/>
        </StackPanel>

        <TextBlock x:Name="StatusText"
                   Grid.Row="1"
                   FontSize="14"
                   Margin="0,0,0,10"
                   Text="Status: Ready" />

        <DataGrid x:Name="AlbumsGrid"
                  Grid.Row="2"
                  AutoGenerateColumns="False"
                  IsReadOnly="True"
                  CanUserAddRows="False">
            <DataGrid.Columns>
                <DataGridTextColumn Header="Id" Binding="{Binding Id}" Width="80"/>
                <DataGridTextColumn Header="UserId" Binding="{Binding UserId}" Width="90"/>
                <DataGridTextColumn Header="Title" Binding="{Binding Title}" Width="*"/>
            </DataGrid.Columns>
        </DataGrid>
    </Grid>
</Window>
```

---

## 7) Model class (Album.cs)

Create a new file named `Album.cs`:

```csharp
namespace Lecture5WebApi
{
    public class Album
    {
        public int UserId { get; set; }
        public int Id { get; set; }
        public string Title { get; set; } = "";
    }
}
```

---

## 8) Code-behind (MainWindow.xaml.cs)

Replace `MainWindow.xaml.cs` with:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;
using System.Windows;

namespace Lecture5WebApi
{
    public partial class MainWindow : Window
    {
        // Reuse one HttpClient instance (good practice for desktop apps)
        private static readonly HttpClient _http = new HttpClient();

        private List<Album> _allAlbums = new List<Album>();

        public MainWindow()
        {
            InitializeComponent();
        }

        private async void LoadButton_Click(object sender, RoutedEventArgs e)
        {
            LoadButton.IsEnabled = false;
            StatusText.Text = "Status: Loading albums...";

            try
            {
                string url = "https://jsonplaceholder.typicode.com/albums";

                HttpResponseMessage response = await _http.GetAsync(url);

                // If not 2xx, throw an exception (we'll catch it)
                response.EnsureSuccessStatusCode();

                string json = await response.Content.ReadAsStringAsync();

                var options = new JsonSerializerOptions
                {
                    PropertyNameCaseInsensitive = true
                };

                List<Album>? albums = JsonSerializer.Deserialize<List<Album>>(json, options);

                _allAlbums = albums ?? new List<Album>();

                AlbumsGrid.ItemsSource = _allAlbums;
                StatusText.Text = $"Status: Loaded {_allAlbums.Count} albums.";
            }
            catch (HttpRequestException ex)
            {
                StatusText.Text = "Status: Network error.";
                MessageBox.Show($"Network error:\n{ex.Message}", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
            }
            catch (JsonException ex)
            {
                StatusText.Text = "Status: JSON parse error.";
                MessageBox.Show($"JSON parse error:\n{ex.Message}", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
            }
            catch (Exception ex)
            {
                StatusText.Text = "Status: Unexpected error.";
                MessageBox.Show($"Unexpected error:\n{ex.Message}", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
            }
            finally
            {
                LoadButton.IsEnabled = true;
            }
        }

        private void SearchBox_TextChanged(object sender, System.Windows.Controls.TextChangedEventArgs e)
        {
            if (_allAlbums.Count == 0)
                return;

            string query = SearchBox.Text.Trim();

            if (string.IsNullOrWhiteSpace(query))
            {
                AlbumsGrid.ItemsSource = _allAlbums;
                StatusText.Text = $"Status: Showing {_allAlbums.Count} albums.";
                return;
            }

            var filtered = _allAlbums
                .Where(a => a.Title.Contains(query, StringComparison.OrdinalIgnoreCase))
                .ToList();

            AlbumsGrid.ItemsSource = filtered;
            StatusText.Text = $"Status: Showing {filtered.Count} filtered albums.";
        }

        private void ClearButton_Click(object sender, RoutedEventArgs e)
        {
            SearchBox.Clear();
            AlbumsGrid.ItemsSource = null;
            _allAlbums.Clear();
            StatusText.Text = "Status: Cleared.";
        }
    }
}
```

### Why this works (GUI-safe)

- The button handler is `async void` (allowed for event handlers)
- We `await` the HTTP call and reading response body
- The UI stays responsive while the request is in progress
- We update the UI after the `await` safely

---

## 9) Important habits (don’t skip)

- **Never** do `.Result` or `.Wait()` in WPF → it can deadlock or freeze.
- Reuse `HttpClient` (don’t create a new one for every request).
- Always handle errors: network failures and JSON parsing failures are normal.
- Keep the UI responsive: `await` all slow operations.

---

## Practice (quick tasks)

1. Add a button “Load Album #1” that calls:

- `https://jsonplaceholder.typicode.com/albums/1`
  and shows it in a MessageBox.

2. Add a column that shows `TitleLength` (computed in code) without changing the API:

- create a property in `Album` like `public int TitleLength => Title.Length;`

3. Add a “Sort by Title” button that sorts the currently displayed list alphabetically.
