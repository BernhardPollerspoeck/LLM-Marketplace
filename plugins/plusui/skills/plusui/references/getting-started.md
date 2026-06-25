# PlusUi Getting Started

Installation, per-platform project setup, the app/host bootstrap, and the minimal MVVM "hello world". PlusUi is a single-codebase, Skia-rendered .NET UI framework with a fluent (builder-pattern) control API and Microsoft.Extensions DI/Hosting.

## Table of contents
- [Packages](#packages)
- [Prerequisites](#prerequisites)
- [Per-platform project setup](#per-platform-project-setup)
- [The App class (IAppConfiguration)](#the-app-class-iappconfiguration)
- [Window configuration options](#window-configuration-options)
- [Minimal hello-world app](#minimal-hello-world-app)
- [Platform entry points (host bootstrap)](#platform-entry-points-host-bootstrap)
- [Builder / DI API](#builder--di-api)
- [Common LLM mistakes](#common-llm-mistakes)

## Packages

`PlusUi.core` is always required. Add exactly one platform "head" package.

| Package | Platform | Head type |
|:--------|:---------|:----------|
| `PlusUi.core` | Core framework (required everywhere) | — |
| `PlusUi.desktop` | Windows / macOS / Linux | `PlusUi.desktop.PlusUiApp` |
| `PlusUi.droid` | Android | `PlusUiActivity` subclass |
| `PlusUi.ios` | iOS | `PlusUiAppDelegate` subclass |
| `PlusUi.web` | Blazor WebAssembly (web/wasm) | `PlusUiWebApp` |
| `PlusUi.headless` | Server-side render / screenshots / tests | `PlusUiHeadless.Create(...)` |
| `PlusUi.h264` | Render UI to an MP4/H264 video | `PlusUi.h264.PlusUiApp` + `IVideoAppConfiguration` |
| `CommunityToolkit.Mvvm` | MVVM (`[ObservableProperty]`, `[RelayCommand]`) — recommended | — |

```bash
dotnet new console -n MyApp && cd MyApp
dotnet add package PlusUi.core
dotnet add package PlusUi.desktop
dotnet add package CommunityToolkit.Mvvm
```

## Prerequisites

- .NET 10 SDK or later. All heads target `net10.0` (Android `net10.0-android`, iOS `net10.0-ios`).
- Use `<LangVersion>preview</LangVersion>` and `<Nullable>enable</Nullable>` (PlusUi relies on primary constructors).
- Linux runtime deps: `libfontconfig1 libfreetype6` (Debian/Ubuntu) or `fontconfig freetype` (Fedora).
- iOS requires macOS + Xcode; Android requires the Android SDK.

## Per-platform project setup

The `.csproj` differs only by SDK, `TargetFramework`/`OutputType`, and the head package.

| Platform | SDK | TargetFramework | OutputType | Head package |
|:---------|:----|:----------------|:-----------|:-------------|
| Windows | `Microsoft.NET.Sdk` | `net10.0` | `WinExe` | `PlusUi.desktop` |
| macOS | `Microsoft.NET.Sdk` | `net10.0` (`RuntimeIdentifier` `osx-arm64`/`osx-x64`) | `Exe` | `PlusUi.desktop` |
| Linux | `Microsoft.NET.Sdk` | `net10.0` | `Exe` | `PlusUi.desktop` |
| Android | `Microsoft.NET.Sdk` | `net10.0-android` | (default) | `PlusUi.droid` |
| iOS | `Microsoft.NET.Sdk` | `net10.0-ios` (`RuntimeIdentifier` `ios-arm64`) | (default) | `PlusUi.ios` |
| Web/Wasm | `Microsoft.NET.Sdk.BlazorWebAssembly` | `net10.0` | (default) | `PlusUi.web` |
| Headless | `Microsoft.NET.Sdk` | `net10.0` | `Exe` | `PlusUi.headless` |
| H264 | `Microsoft.NET.Sdk` | `net10.0` | `Exe` | `PlusUi.h264` |

Desktop (Windows) `.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <LangVersion>preview</LangVersion>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="PlusUi.core" Version="*" />
    <PackageReference Include="PlusUi.desktop" Version="*" />
  </ItemGroup>
</Project>
```

For Web swap to `<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">` + `PlusUi.web`; Android uses `net10.0-android` + `PlusUi.droid`; iOS uses `net10.0-ios` + `RuntimeIdentifier ios-arm64` + `PlusUi.ios`.

**Multi-platform layout (recommended):** put pages/viewmodels/`App.cs` in a shared `MyApp.Core` library (`PlusUi.core` only, plain `net10.0`); each platform head is a thin project that `ProjectReference`s `MyApp.Core` and adds its platform package + entry point.

## The App class (IAppConfiguration)

Every app (desktop, mobile, web, headless) shares ONE `App : IAppConfiguration`. The platform head only differs in how it launches that App.

```csharp
public interface IAppConfiguration
{
    void ConfigureWindow(PlusUiConfiguration configuration);   // window/runtime settings
    void ConfigureApp(IPlusUiAppBuilder builder);              // register pages, VMs, services, style
    UiPageElement GetRootPage(IServiceProvider serviceProvider); // first page to show
}
```

- `ConfigureApp` receives `IPlusUiAppBuilder` (exposes `.Services` and the fluent `AddPage`/`WithViewModel`/... extensions). It is NOT the raw `HostApplicationBuilder`.
- `GetRootPage` resolves the page from DI, e.g. `serviceProvider.GetRequiredService<MainPage>()`.

## Window configuration options

Set on `PlusUiConfiguration` inside `ConfigureWindow`. Common options:

| Property | Type | Default | Notes |
|:---------|:-----|:--------|:------|
| `Title` | `string` | `"Plus Ui Application"` | Window title |
| `Size` | `SizeI` | `800x600` | Initial window/frame size; headless reads this as render size |
| `Position` | `SizeI` | `100x100` | Initial window position (desktop) |
| `EnableNavigationStack` | `bool` | `false` | Enables `GoBack` / push-pop navigation |
| `PreservePageState` | `bool` | `true` | Keep page state across navigation |
| `RememberWindowPosition` | `bool` | `false` | Persist window pos/size (desktop) |
| `WindowIcon` | `string?` | `null` | Icon resource (e.g. `"plusui.svg"`) |
| `ApplicationId` | `string?` | `null` | App identifier for settings persistence |
| `LoadImagesSynchronously` | `bool` | `false` | Useful for headless/h264 deterministic render |
| `WindowState` / `WindowBorder` | enum | `Normal` / `Resizable` | Desktop only |
| `IsWindowTransparent` / `IsWindowTopMost` | `bool` | `false` | Desktop only |
| `EnableHighContrastSupport` / `RespectReducedMotion` / `EnableFontScaling` | `bool` | `false` | Accessibility |
| `DefaultTransition` | `IPageTransition` | `SlideTransition` | Global page transition |

## Minimal hello-world app

Four files. Pages and ViewModels are plain C# classes built with the fluent API — there is no XAML.

**`App.cs`** — shared configuration:

```csharp
using Microsoft.Extensions.DependencyInjection;
using PlusUi.core;

namespace MyApp;

public class App : IAppConfiguration
{
    public void ConfigureWindow(PlusUiConfiguration configuration)
    {
        configuration.Title = "My First PlusUi App";
        configuration.Size = new SizeI(800, 600);
    }

    public void ConfigureApp(IPlusUiAppBuilder builder)
    {
        builder.AddPage<MainPage>().WithViewModel<MainPageViewModel>();
        // builder.StylePlusUi<MyCustomStyle>();   // optional global style
    }

    public UiPageElement GetRootPage(IServiceProvider serviceProvider)
        => serviceProvider.GetRequiredService<MainPage>();
}
```

**`MainPageViewModel.cs`** — MVVM via CommunityToolkit source generators:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace MyApp;

public partial class MainPageViewModel : ObservableObject
{
    [ObservableProperty]
    private string _greeting = "Hello, PlusUi!";

    [ObservableProperty]
    private int _clickCount;

    [RelayCommand]
    private void IncrementCounter()
    {
        ClickCount++;
        Greeting = $"Clicked {ClickCount} times!";
    }
}
```

**`MainPage.cs`** — the VM is injected via primary constructor; `Build()` returns the tree:

```csharp
using PlusUi.core;

namespace MyApp;

public class MainPage(MainPageViewModel vm) : UiPageElement(vm)
{
    protected override UiElement Build()
    {
        return new VStack(
            new Label()
                .BindText(() => vm.Greeting)
                .SetTextSize(32)
                .SetTextColor(Colors.White)
                .SetHorizontalAlignment(HorizontalAlignment.Center),

            new Button()
                .SetText("Click Me!")
                .SetPadding(new Margin(20, 10))
                .SetCommand(vm.IncrementCounterCommand)
                .SetHorizontalAlignment(HorizontalAlignment.Center)
        )
        .SetVerticalAlignment(VerticalAlignment.Center);
    }
}
```

Rules for pages/VMs:
- A page MUST inherit `UiPageElement` and pass the VM to the base ctor: `MainPage(MainPageViewModel vm) : UiPageElement(vm)`.
- Override `protected override UiElement Build()` and return a single root element (use a layout like `VStack`/`Grid` to host multiple children).
- `[RelayCommand] IncrementCounter` generates the `IncrementCounterCommand` property — bind it with `.SetCommand(vm.IncrementCounterCommand)`.
- Every `Set*` has a matching `Bind*`. For one-way binding to a VM property use `.BindText(() => vm.Greeting)` (the expression both names the property to observe and reads its current value).

## Platform entry points (host bootstrap)

The App class is identical across all of these; only the launcher changes.

**Desktop** (`Program.cs`):

```csharp
using PlusUi.desktop;
using MyApp;

var app = new PlusUiApp(args);
app.CreateApp(builder => new App());   // builder is HostApplicationBuilder
```

**Web / Wasm** (`Program.cs`):

```csharp
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using PlusUi.Web;
using MyApp;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
var app = new PlusUiWebApp(builder);
await app.CreateApp(() => new App());
```

**Android** (`MainActivity.cs`):

```csharp
using Microsoft.Extensions.Hosting;
using PlusUi.core;
using PlusUi.droid;
using MyApp;

namespace MyApp.Android;

[Activity(Label = "@string/app_name", MainLauncher = true)]
public class MainActivity : PlusUiActivity
{
    protected override IAppConfiguration CreateApp(HostApplicationBuilder builder) => new App();
}
```

**iOS** (`AppDelegate.cs`; `Main.cs` calls `UIApplication.Main(args, null, typeof(AppDelegate))`):

```csharp
using Microsoft.Extensions.Hosting;
using PlusUi.core;
using PlusUi.ios;
using MyApp;

namespace MyApp.ios;

[Register("AppDelegate")]
public class AppDelegate : PlusUiAppDelegate
{
    protected override IAppConfiguration CreateApp(HostApplicationBuilder builder) => new App();
}
```

**Headless** (`Program.cs`) — renders a frame without a window/display:

```csharp
using PlusUi.Headless;
using PlusUi.Headless.Enumerations;
using MyApp;

using var headless = PlusUiHeadless.Create(new App(), ImageFormat.Png);
var frame = await headless.GetCurrentFrameAsync();
await File.WriteAllBytesAsync("screenshot.png", frame);
```

Headless reads `PlusUiConfiguration.Size` as the render resolution. Each `Create(...)` instance is fully isolated (own ServiceProvider) and disposable. Set `LoadImagesSynchronously = true` so images are ready before the frame is captured.

**H264 video** — implement `IVideoAppConfiguration` (not `IAppConfiguration`) and launch with `PlusUi.h264.PlusUiApp`:

```csharp
using PlusUi.h264;
using MyApp;

var app = new PlusUiApp(args);
app.CreateApp(builder => new VideoApp());   // VideoApp : IVideoAppConfiguration
```

```csharp
public class VideoApp : IVideoAppConfiguration
{
    public void ConfigureApp(IPlusUiAppBuilder builder)
    {
        builder.Services.AddSingleton<MainPageViewModel>();
        builder.Services.AddSingleton<MainPage>();
        builder.Services.AddSingleton<IApplicationStyle, ApplicationStyle>();
    }

    public UiPageElement GetRootPage(IServiceProvider sp)
        => sp.GetRequiredService<MainPage>();

    public IAudioSequenceProvider? GetAudioSequenceProvider(IServiceProvider sp) => null;
    public IVideoOverlayProvider? GetVideoOverlayProvider(IServiceProvider sp) => null;

    public void ConfigureVideo(VideoConfiguration v)
    {
        v.Width = 800; v.Height = 450;
        v.OutputFilePath = "../output.mp4";
        v.FrameRate = 60;
        v.Duration = TimeSpan.FromSeconds(11);
    }
}
```

## Builder / DI API

PlusUi uses `Microsoft.Extensions.DependencyInjection`. Fluent extensions on `IPlusUiAppBuilder`:

| Method | Effect |
|:-------|:-------|
| `AddPage<TPage>()` | Registers a `UiPageElement` page as transient |
| `.WithViewModel<TVm>()` | Registers a VM (`INotifyPropertyChanged`) as transient; chainable after `AddPage` |
| `AddPopup<TPopup>()` | Registers a `UiPopupElement` as transient |
| `StylePlusUi<TStyle>()` | Registers a global `IApplicationStyle` singleton |
| `RegisterFont(resourcePath, fontFamily, weight?, style?)` | Registers a custom font |
| `builder.Services` | Raw `IServiceCollection` for your own services |

```csharp
public void ConfigureApp(IPlusUiAppBuilder builder)
{
    builder.AddPage<MainPage>().WithViewModel<MainPageViewModel>();
    builder.AddPage<SettingsPage>().WithViewModel<SettingsPageViewModel>();
    builder.AddPopup<ConfirmPopup>().WithViewModel<ConfirmPopupViewModel>();
    builder.StylePlusUi<DefaultStyle>();
    builder.Services.AddSingleton<IMyService, MyService>();
}
```

- VMs and services are injected via **primary constructors**: `public partial class MyViewModel(INavigationService nav, IMyService svc) : ObservableObject`.
- Navigation (when `EnableNavigationStack = true`): inject `INavigationService`, call `nav.NavigateTo<DetailPage>()` / `nav.NavigateTo<DetailPage>(param)` / `nav.GoBack()`. Receive params by implementing `INavigationAware.OnNavigatedTo(object?)` on the target VM.
- Popups: inject `IPopupService`, `await popupService.ShowPopup<ConfirmPopup, bool>()`.
- A page with no state can be registered VM-less (`builder.AddPage<LabelPage>()`); or register a shared VM once with `builder.WithViewModel<DemoPageViewModel>()` and reuse it.

## Common LLM mistakes

- **`ConfigureApp` parameter type.** It is `IPlusUiAppBuilder`, not `HostApplicationBuilder`. Use `builder.Services` / the fluent extensions. (Only the platform-head `CreateApp(...)` callback receives the raw `HostApplicationBuilder`.)
- **No XAML.** Pages are C# classes overriding `UiElement Build()`. Do not invent `.xaml` / `InitializeComponent()`.
- **Page base ctor.** `MainPage(MainPageViewModel vm) : UiPageElement(vm)` — forgetting to pass the VM to the base breaks data binding.
- **Single root from `Build()`.** Wrap multiple children in a layout (`VStack`, `HStack`, `Grid`); `Build()` returns one `UiElement`.
- **Command naming.** `[RelayCommand] private void Foo()` generates `FooCommand`; bind with `.SetCommand(vm.FooCommand)`, not `vm.Foo`.
- **`BindText` signature.** `BindText` requires an expression, not `nameof`. For string properties use `.BindText(() => vm.X)`. For non-string types use `.BindText(() => vm.X, x => x.ToString())` to provide a formatter.
- **Wrong head per platform.** Desktop/h264 launch via `new PlusUiApp(args).CreateApp(...)`; Web via `PlusUiWebApp.CreateApp` (async, `() => new App()`); Android/iOS via subclassing `PlusUiActivity`/`PlusUiAppDelegate`; headless via static `PlusUiHeadless.Create(...)`.
- **H264 uses a different interface.** `IVideoAppConfiguration` (with `ConfigureVideo`, `GetVideoOverlayProvider`, `GetAudioSequenceProvider`), not `IAppConfiguration`.
- **Target framework.** All heads need .NET 10 + `LangVersion preview`; primary constructors and source generators will not compile otherwise.
- **One App, many heads.** Don't duplicate page/VM registration per platform — keep `App.cs` shared and let each head just return `new App()`.
