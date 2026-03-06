# Lecture 6 — Firebase Database and Authentication (WPF + C#)

**Goal:** Build a WPF desktop app that lets users **Sign Up / Log In (Email+Password)**, **Log In with Google**, then perform **CRUD** on data stored in **Firebase Realtime Database**.

---

## 1) What you’re building (concept)

**WPF App**

- Auth screen:
  - Email+Password: **Sign Up**, **Sign In**
  - **Sign in with Google** (OAuth → Google ID Token → Firebase)
- After login:
  - A simple “Tasks” screen (DataGrid + Add/Update/Delete)

**Firebase**

- **Firebase Authentication**
  - Email/Password
  - Google provider :contentReference[oaicite:2]{index=2}
- **Realtime Database**
  - Store user data under:
    - `/tasks/{uid}/{taskKey}`

**Important**

- Realtime Database REST requests can be authenticated using a **Firebase ID token** via `?auth=<ID_TOKEN>`.
- Don’t put **service account credentials** in a client app.

---

## 2) Firebase Console setup (must do once)

### A) Create project + enable Authentication providers

1. Firebase Console → **Authentication** → **Sign-in method**
2. Enable:
   - **Email/Password**
   - **Google**

### B) Create a Web App (to get API key)

Firebase Console → Project settings → General → Add app → **Web App**  
Copy the **Web API Key**.

> Firebase API keys identify your project; they are not “secret keys”, but you should still apply best practices (restrict where possible).

### C) Create Realtime Database

Firebase Console → **Realtime Database** → Create database

### D) Set Realtime Database rules (basic per-user isolation)

Realtime Database → Rules:

```json
{
  "rules": {
    "tasks": {
      "$uid": {
        ".read": "auth != null && auth.uid == $uid",
        ".write": "auth != null && auth.uid == $uid"
      }
    }
  }
}
```

This forces each user to only access their own data.

---

## 3) Endpoints you’ll use (REST)

### Firebase Auth REST (Identity Toolkit / Identity Platform)

Base: `https://identitytoolkit.googleapis.com`

- **Sign Up (Email/Pass)**
  `POST https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=API_KEY`
- **Sign In (Email/Pass)**
  `POST https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=API_KEY`

- **Sign In with Google (ID token → Firebase)**
  `POST https://identitytoolkit.googleapis.com/v1/accounts:signInWithIdp?key=API_KEY`
  `postBody: id_token=<GOOGLE_ID_TOKEN>&providerId=google.com`

### Realtime Database REST

- Base: `https://YOUR_DB_NAME.firebaseio.com/`
- Endpoints must end in `.json` ([Firebase][5])
- Auth token can be passed as `?auth=<FIREBASE_ID_TOKEN>`
- Save data via REST with `PUT/PATCH/POST/DELETE`

---

## 4) WPF App UI (simple, Lecture 4 style)

Create project:

```bash
dotnet new wpf -n Lecture6Firebase
```

### `MainWindow.xaml`

A minimal “2-panel” layout:

- Left: Authentication
- Right: CRUD tasks UI

```xml
<Window x:Class="Lecture6Firebase.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Lecture 6 - Firebase" Height="540" Width="980">
    <Grid Margin="16">
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="320"/>
            <ColumnDefinition Width="20"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>

        <!-- AUTH PANEL -->
        <Border BorderBrush="Gray" BorderThickness="1" Padding="12" CornerRadius="6">
            <StackPanel>
                <TextBlock Text="Authentication" FontSize="18" Margin="0,0,0,10"/>

                <TextBlock Text="Email"/>
                <TextBox x:Name="EmailBox" Height="30" Margin="0,4,0,10"/>

                <TextBlock Text="Password"/>
                <PasswordBox x:Name="PasswordBox" Height="30" Margin="0,4,0,10"/>

                <StackPanel Orientation="Horizontal">
                    <Button x:Name="SignUpButton" Content="Sign Up" Width="90" Height="32" Click="SignUpButton_Click"/>
                    <Button x:Name="SignInButton" Content="Sign In" Width="90" Height="32" Margin="10,0,0,0" Click="SignInButton_Click"/>
                </StackPanel>

                <Button x:Name="GoogleButton" Content="Sign In with Google"
                        Height="32" Margin="0,10,0,0"
                        Click="GoogleButton_Click"/>

                <Button x:Name="SignOutButton" Content="Sign Out"
                        Height="32" Margin="0,10,0,0"
                        Click="SignOutButton_Click"/>

                <TextBlock Text="Status" Margin="0,14,0,0"/>
                <TextBlock x:Name="AuthStatusText" Text="Ready" TextWrapping="Wrap" Margin="0,4,0,0"/>

                <TextBlock Text="User (UID)" Margin="0,14,0,0"/>
                <TextBlock x:Name="UidText" Text="-" TextWrapping="Wrap" Margin="0,4,0,0"/>
            </StackPanel>
        </Border>

        <!-- TASKS PANEL -->
        <Border Grid.Column="2" BorderBrush="Gray" BorderThickness="1" Padding="12" CornerRadius="6">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>

                <TextBlock Text="Tasks (CRUD in Firebase Realtime Database)" FontSize="18" Margin="0,0,0,10"/>

                <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,0,0,10">
                    <TextBox x:Name="TitleBox" Width="360" Height="30" VerticalContentAlignment="Center" ToolTip="Task title"/>
                    <CheckBox x:Name="CompletedBox" Content="Completed" Margin="10,0,0,0" VerticalAlignment="Center"/>

                    <Button x:Name="AddButton" Content="Add" Width="80" Height="30" Margin="10,0,0,0" Click="AddButton_Click"/>
                    <Button x:Name="UpdateButton" Content="Update" Width="80" Height="30" Margin="10,0,0,0" Click="UpdateButton_Click"/>
                    <Button x:Name="DeleteButton" Content="Delete" Width="80" Height="30" Margin="10,0,0,0" Click="DeleteButton_Click"/>
                    <Button x:Name="RefreshButton" Content="Refresh" Width="80" Height="30" Margin="10,0,0,0" Click="RefreshButton_Click"/>
                </StackPanel>

                <DataGrid x:Name="TasksGrid" Grid.Row="2"
                          AutoGenerateColumns="False"
                          IsReadOnly="True"
                          CanUserAddRows="False"
                          SelectionChanged="TasksGrid_SelectionChanged">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="Key" Binding="{Binding Key}" Width="220"/>
                        <DataGridTextColumn Header="Title" Binding="{Binding Title}" Width="*"/>
                        <DataGridCheckBoxColumn Header="Completed" Binding="{Binding Completed}" Width="110"/>
                        <DataGridTextColumn Header="Created" Binding="{Binding CreatedAt}" Width="170"/>
                    </DataGrid.Columns>
                </DataGrid>
            </Grid>
        </Border>
    </Grid>
</Window>
```

---

## 5) Code files (copy/paste)

### A) `FirebaseConfig.cs`

Fill these from your Firebase console + Google Cloud OAuth.

```csharp
namespace Lecture6Firebase;

public static class FirebaseConfig
{
    // Firebase Project Settings -> Web API Key
    public const string ApiKey = "PASTE_YOUR_FIREBASE_WEB_API_KEY";

    // Realtime Database URL, e.g. https://your-db-name.firebaseio.com
    public const string DatabaseUrl = "https://YOUR_DB_NAME.firebaseio.com";

    // Google OAuth Client ID (type: Desktop app)
    public const string GoogleClientId = "PASTE_YOUR_GOOGLE_OAUTH_CLIENT_ID.apps.googleusercontent.com";
}
```

---

### B) Models: `FirebaseAuthSession.cs` and `TaskItem.cs`

```csharp
namespace Lecture6Firebase;

public class FirebaseAuthSession
{
    public string IdToken { get; set; } = "";
    public string RefreshToken { get; set; } = "";
    public string LocalId { get; set; } = "";   // uid
    public string Email { get; set; } = "";
    public int ExpiresIn { get; set; }          // seconds
}
```

```csharp
namespace Lecture6Firebase;

public class TaskItem
{
    // Firebase key (generated by POST)
    public string Key { get; set; } = "";

    public string Title { get; set; } = "";
    public bool Completed { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

---

### C) `FirebaseAuthService.cs` (Email/Pass + Google)

Notes:

- Email/password sign-up endpoint
- Email/password sign-in endpoint
- Google sign-in via `signInWithIdp` expects a Google ID token in `postBody`

```csharp
using System.Net.Http;
using System.Text;
using System.Text.Json;

namespace Lecture6Firebase;

public class FirebaseAuthService
{
    private static readonly HttpClient _http = new HttpClient();

    public async Task<FirebaseAuthSession> SignUpAsync(string email, string password)
    {
        var url = $"https://identitytoolkit.googleapis.com/v1/accounts:signUp?key={FirebaseConfig.ApiKey}";

        var payload = new
        {
            email,
            password,
            returnSecureToken = true
        };

        return await PostAuthAsync(url, payload);
    }

    public async Task<FirebaseAuthSession> SignInAsync(string email, string password)
    {
        var url = $"https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={FirebaseConfig.ApiKey}";

        var payload = new
        {
            email,
            password,
            returnSecureToken = true
        };

        return await PostAuthAsync(url, payload);
    }

    public async Task<FirebaseAuthSession> SignInWithGoogleIdTokenAsync(string googleIdToken)
    {
        var url = $"https://identitytoolkit.googleapis.com/v1/accounts:signInWithIdp?key={FirebaseConfig.ApiKey}";

        // postBody format documented in signInWithIdp:
        // id_token=<GOOGLE_ID_TOKEN>&providerId=google.com :contentReference[oaicite:17]{index=17}
        string postBody = $"id_token={Uri.EscapeDataString(googleIdToken)}&providerId=google.com";

        var payload = new
        {
            postBody,
            requestUri = "http://localhost", // required
            returnSecureToken = true,
            returnIdpCredential = true
        };

        return await PostAuthAsync(url, payload);
    }

    private static async Task<FirebaseAuthSession> PostAuthAsync(string url, object payload)
    {
        string json = JsonSerializer.Serialize(payload);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        using HttpResponseMessage resp = await _http.PostAsync(url, content);
        string respJson = await resp.Content.ReadAsStringAsync();

        if (!resp.IsSuccessStatusCode)
            throw new Exception(respJson);

        using var doc = JsonDocument.Parse(respJson);
        var root = doc.RootElement;

        // Firebase returns:
        // idToken, refreshToken, localId, email, expiresIn
        return new FirebaseAuthSession
        {
            IdToken = root.GetProperty("idToken").GetString() ?? "",
            RefreshToken = root.GetProperty("refreshToken").GetString() ?? "",
            LocalId = root.GetProperty("localId").GetString() ?? "",
            Email = root.TryGetProperty("email", out var e) ? (e.GetString() ?? "") : "",
            ExpiresIn = int.TryParse(root.GetProperty("expiresIn").GetString(), out var secs) ? secs : 0
        };
    }
}
```

---

### D) `FirebaseDatabaseService.cs` (CRUD via Realtime Database REST)

- Append `.json` to endpoints ([Firebase][5])
- Use `POST/PATCH/DELETE` etc. ([Firebase][7])
- Authenticate via `?auth=<ID_TOKEN>` ([Firebase][6])

```csharp
using System.Net.Http;
using System.Text;
using System.Text.Json;

namespace Lecture6Firebase;

public class FirebaseDatabaseService
{
    private static readonly HttpClient _http = new HttpClient();

    private static string BaseTasksUrl(string uid) =>
        $"{FirebaseConfig.DatabaseUrl.TrimEnd('/')}/tasks/{uid}";

    public async Task<List<TaskItem>> GetTasksAsync(string uid, string idToken)
    {
        string url = $"{BaseTasksUrl(uid)}.json?auth={Uri.EscapeDataString(idToken)}";

        using HttpResponseMessage resp = await _http.GetAsync(url);
        string json = await resp.Content.ReadAsStringAsync();

        if (!resp.IsSuccessStatusCode)
            throw new Exception(json);

        if (json == "null")
            return new List<TaskItem>();

        var dict = JsonSerializer.Deserialize<Dictionary<string, TaskItemDto>>(json) ?? new();

        return dict.Select(kvp => new TaskItem
        {
            Key = kvp.Key,
            Title = kvp.Value.Title ?? "",
            Completed = kvp.Value.Completed,
            CreatedAt = DateTimeOffset.FromUnixTimeSeconds(kvp.Value.CreatedAtUnix).LocalDateTime
        })
        .OrderByDescending(t => t.CreatedAt)
        .ToList();
    }

    public async Task<string> AddTaskAsync(string uid, string idToken, string title, bool completed)
    {
        string url = $"{BaseTasksUrl(uid)}.json?auth={Uri.EscapeDataString(idToken)}";

        var dto = new TaskItemDto
        {
            Title = title,
            Completed = completed,
            CreatedAtUnix = DateTimeOffset.UtcNow.ToUnixTimeSeconds()
        };

        string json = JsonSerializer.Serialize(dto);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        using HttpResponseMessage resp = await _http.PostAsync(url, content);
        string respJson = await resp.Content.ReadAsStringAsync();

        if (!resp.IsSuccessStatusCode)
            throw new Exception(respJson);

        // POST returns { "name": "<generatedKey>" }
        using var doc = JsonDocument.Parse(respJson);
        return doc.RootElement.GetProperty("name").GetString() ?? "";
    }

    public async Task UpdateTaskAsync(string uid, string idToken, string key, string title, bool completed)
    {
        string url = $"{BaseTasksUrl(uid)}/{key}.json?auth={Uri.EscapeDataString(idToken)}";

        // PATCH only the fields we want to update
        var patch = new
        {
            title,
            completed
        };

        var req = new HttpRequestMessage(new HttpMethod("PATCH"), url)
        {
            Content = new StringContent(JsonSerializer.Serialize(patch), Encoding.UTF8, "application/json")
        };

        using HttpResponseMessage resp = await _http.SendAsync(req);
        string respJson = await resp.Content.ReadAsStringAsync();

        if (!resp.IsSuccessStatusCode)
            throw new Exception(respJson);
    }

    public async Task DeleteTaskAsync(string uid, string idToken, string key)
    {
        string url = $"{BaseTasksUrl(uid)}/{key}.json?auth={Uri.EscapeDataString(idToken)}";

        using HttpResponseMessage resp = await _http.DeleteAsync(url);
        string respJson = await resp.Content.ReadAsStringAsync();

        if (!resp.IsSuccessStatusCode)
            throw new Exception(respJson);
    }

    private class TaskItemDto
    {
        public string? Title { get; set; }
        public bool Completed { get; set; }
        public long CreatedAtUnix { get; set; }
    }
}
```

---

## 6) Google Login (OAuth → Google ID Token)

### The correct mental model

- Google Sign-In (native app) should use the system browser (best practice per OAuth for native apps)
- You request `openid email profile` to receive an **ID token** (OpenID Connect)
- Then you exchange that Google ID token with Firebase using `accounts:signInWithIdp`

### `GoogleOAuthHelper.cs` (PKCE + loopback redirect)

Create a Desktop OAuth Client ID in Google Cloud console (type “Desktop app”), then paste into `FirebaseConfig.GoogleClientId`.

```csharp
using System.Diagnostics;
using System.Net;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

namespace Lecture6Firebase;

public static class GoogleOAuthHelper
{
    public static async Task<string> GetGoogleIdTokenAsync()
    {
        // Pick a localhost port
        int port = GetFreeTcpPort();
        string redirectUri = $"http://127.0.0.1:{port}/";

        string codeVerifier = CreateCodeVerifier();
        string codeChallenge = CreateCodeChallenge(codeVerifier);

        string scope = Uri.EscapeDataString("openid email profile");
        string authUrl =
            "https://accounts.google.com/o/oauth2/v2/auth" +
            $"?client_id={Uri.EscapeDataString(FirebaseConfig.GoogleClientId)}" +
            $"&redirect_uri={Uri.EscapeDataString(redirectUri)}" +
            $"&response_type=code" +
            $"&scope={scope}" +
            $"&code_challenge={Uri.EscapeDataString(codeChallenge)}" +
            $"&code_challenge_method=S256";

        using var listener = new HttpListener();
        listener.Prefixes.Add(redirectUri);
        listener.Start();

        Process.Start(new ProcessStartInfo(authUrl) { UseShellExecute = true });

        // Wait for browser redirect with ?code=...
        var context = await listener.GetContextAsync();
        string? code = context.Request.QueryString["code"];

        // Respond to browser so user sees something
        string responseHtml = "<html><body><h2>You can close this window.</h2></body></html>";
        byte[] buffer = Encoding.UTF8.GetBytes(responseHtml);
        context.Response.ContentLength64 = buffer.Length;
        await context.Response.OutputStream.WriteAsync(buffer);
        context.Response.OutputStream.Close();

        if (string.IsNullOrWhiteSpace(code))
            throw new Exception("Google OAuth did not return an authorization code.");

        // Exchange code for tokens (includes id_token)
        using var http = new HttpClient();

        var tokenRequest = new Dictionary<string, string>
        {
            ["client_id"] = FirebaseConfig.GoogleClientId,
            ["code"] = code,
            ["code_verifier"] = codeVerifier,
            ["redirect_uri"] = redirectUri,
            ["grant_type"] = "authorization_code"
        };

        using var content = new FormUrlEncodedContent(tokenRequest);
        using var resp = await http.PostAsync("https://oauth2.googleapis.com/token", content);
        string json = await resp.Content.ReadAsStringAsync();

        if (!resp.IsSuccessStatusCode)
            throw new Exception(json);

        using var doc = JsonDocument.Parse(json);
        string idToken = doc.RootElement.GetProperty("id_token").GetString() ?? "";

        if (string.IsNullOrWhiteSpace(idToken))
            throw new Exception("No id_token returned by Google.");

        return idToken;
    }

    private static int GetFreeTcpPort()
    {
        var l = new System.Net.Sockets.TcpListener(IPAddress.Loopback, 0);
        l.Start();
        int port = ((IPEndPoint)l.LocalEndpoint).Port;
        l.Stop();
        return port;
    }

    private static string CreateCodeVerifier()
    {
        byte[] bytes = RandomNumberGenerator.GetBytes(32);
        return Base64UrlEncode(bytes);
    }

    private static string CreateCodeChallenge(string verifier)
    {
        byte[] bytes = SHA256.HashData(Encoding.ASCII.GetBytes(verifier));
        return Base64UrlEncode(bytes);
    }

    private static string Base64UrlEncode(byte[] bytes)
    {
        return Convert.ToBase64String(bytes)
            .TrimEnd('=')
            .Replace('+', '-')
            .Replace('/', '_');
    }
}
```

---

## 7) Wire it all together in WPF (`MainWindow.xaml.cs`)

```csharp
using System.Collections.ObjectModel;
using System.Windows;

namespace Lecture6Firebase;

public partial class MainWindow : Window
{
    private readonly FirebaseAuthService _auth = new();
    private readonly FirebaseDatabaseService _db = new();

    private FirebaseAuthSession? _session;
    private ObservableCollection<TaskItem> _tasks = new();

    public MainWindow()
    {
        InitializeComponent();
        TasksGrid.ItemsSource = _tasks;
        SetUiEnabled(false);
        AuthStatusText.Text = "Ready";
    }

    // ---------- AUTH ----------
    private async void SignUpButton_Click(object sender, RoutedEventArgs e)
    {
        try
        {
            AuthStatusText.Text = "Signing up...";
            _session = await _auth.SignUpAsync(EmailBox.Text.Trim(), PasswordBox.Password);
            ApplySession(_session);
            AuthStatusText.Text = $"Signed up: {_session.Email}";
            await RefreshTasksAsync();
        }
        catch (Exception ex)
        {
            AuthStatusText.Text = "Sign up failed.";
            MessageBox.Show(ex.Message, "Error");
        }
    }

    private async void SignInButton_Click(object sender, RoutedEventArgs e)
    {
        try
        {
            AuthStatusText.Text = "Signing in...";
            _session = await _auth.SignInAsync(EmailBox.Text.Trim(), PasswordBox.Password);
            ApplySession(_session);
            AuthStatusText.Text = $"Signed in: {_session.Email}";
            await RefreshTasksAsync();
        }
        catch (Exception ex)
        {
            AuthStatusText.Text = "Sign in failed.";
            MessageBox.Show(ex.Message, "Error");
        }
    }

    private async void GoogleButton_Click(object sender, RoutedEventArgs e)
    {
        try
        {
            AuthStatusText.Text = "Opening browser for Google login...";

            string googleIdToken = await GoogleOAuthHelper.GetGoogleIdTokenAsync();
            _session = await _auth.SignInWithGoogleIdTokenAsync(googleIdToken);

            ApplySession(_session);
            AuthStatusText.Text = $"Signed in with Google: {_session.Email}";
            await RefreshTasksAsync();
        }
        catch (Exception ex)
        {
            AuthStatusText.Text = "Google sign-in failed.";
            MessageBox.Show(ex.Message, "Error");
        }
    }

    private void SignOutButton_Click(object sender, RoutedEventArgs e)
    {
        _session = null;
        UidText.Text = "-";
        _tasks.Clear();
        SetUiEnabled(false);
        AuthStatusText.Text = "Signed out.";
    }

    private void ApplySession(FirebaseAuthSession session)
    {
        UidText.Text = session.LocalId;
        SetUiEnabled(true);
    }

    private void SetUiEnabled(bool signedIn)
    {
        AddButton.IsEnabled = signedIn;
        UpdateButton.IsEnabled = signedIn;
        DeleteButton.IsEnabled = signedIn;
        RefreshButton.IsEnabled = signedIn;
    }

    // ---------- CRUD ----------
    private async void RefreshButton_Click(object sender, RoutedEventArgs e)
    {
        await RefreshTasksAsync();
    }

    private async Task RefreshTasksAsync()
    {
        if (_session == null) return;

        try
        {
            var items = await _db.GetTasksAsync(_session.LocalId, _session.IdToken);
            _tasks = new ObservableCollection<TaskItem>(items);
            TasksGrid.ItemsSource = _tasks;
        }
        catch (Exception ex)
        {
            MessageBox.Show(ex.Message, "DB Error");
        }
    }

    private async void AddButton_Click(object sender, RoutedEventArgs e)
    {
        if (_session == null) return;

        string title = TitleBox.Text.Trim();
        if (string.IsNullOrWhiteSpace(title)) return;

        try
        {
            await _db.AddTaskAsync(_session.LocalId, _session.IdToken, title, CompletedBox.IsChecked == true);
            TitleBox.Clear();
            CompletedBox.IsChecked = false;
            await RefreshTasksAsync();
        }
        catch (Exception ex)
        {
            MessageBox.Show(ex.Message, "DB Error");
        }
    }

    private async void UpdateButton_Click(object sender, RoutedEventArgs e)
    {
        if (_session == null) return;
        if (TasksGrid.SelectedItem is not TaskItem selected) return;

        string title = TitleBox.Text.Trim();
        if (string.IsNullOrWhiteSpace(title)) return;

        try
        {
            await _db.UpdateTaskAsync(_session.LocalId, _session.IdToken, selected.Key, title, CompletedBox.IsChecked == true);
            await RefreshTasksAsync();
        }
        catch (Exception ex)
        {
            MessageBox.Show(ex.Message, "DB Error");
        }
    }

    private async void DeleteButton_Click(object sender, RoutedEventArgs e)
    {
        if (_session == null) return;
        if (TasksGrid.SelectedItem is not TaskItem selected) return;

        try
        {
            await _db.DeleteTaskAsync(_session.LocalId, _session.IdToken, selected.Key);
            TitleBox.Clear();
            CompletedBox.IsChecked = false;
            await RefreshTasksAsync();
        }
        catch (Exception ex)
        {
            MessageBox.Show(ex.Message, "DB Error");
        }
    }

    private void TasksGrid_SelectionChanged(object sender, System.Windows.Controls.SelectionChangedEventArgs e)
    {
        if (TasksGrid.SelectedItem is not TaskItem selected) return;
        TitleBox.Text = selected.Title;
        CompletedBox.IsChecked = selected.Completed;
    }
}
```

---

## 8) Token notes (what students should understand)

- Firebase returns an **ID token** and **refresh token** when signing in. ID tokens are short-lived (~1 hour) and refresh tokens allow obtaining new ID tokens (sessions are long-lived).
- For this lecture, you can ignore refresh, but in a real app you’d store refresh tokens securely and refresh when needed.

---

## 9) Common failure points (and how to fix fast)

- **401 / Permission denied** on database calls:
  - Your DB rules are too strict or you’re not passing `?auth=<idToken>`

- **Google sign-in returns but Firebase fails**:
  - Google provider not enabled in Firebase Auth
  - Wrong OAuth client ID

- **UI freezes**:
  - You used `.Result` or `.Wait()` somewhere (don’t). Use `await`.
