# PlusUi Navigation & App Structure Reference

How pages are registered, resolved, and navigated in PlusUi. Navigation is page-based: a single `NavigationContainer` holds a stack of `UiPageElement` instances; `INavigationService` pushes/pops them; pages bind to a ViewModel via constructor DI.

## Table of contents

- [App structure: IAppConfiguration](#app-structure-iappconfiguration)
- [Registering pages & view models](#registering-pages--view-models)
- [The root page & startup](#the-root-page--startup)
- [INavigationService API](#inavigationservice-api)
- [Navigating with parameters](#navigating-with-parameters)
- [Page <-> ViewModel relationship](#page---viewmodel-relationship)
- [Page lifecycle](#page-lifecycle)
- [Navigation configuration (PlusUiConfiguration)](#navigation-configuration-plusuiconfiguration)
- [Navigation stack internals](#navigation-stack-internals)
- [PageChanged event](#pagechanged-event)
- [Real example: master/detail flow](#real-example-masterdetail-flow)
- [Common LLM mistakes](#common-llm-mistakes)

---

## App structure: IAppConfiguration

Every PlusUi app provides one `IAppConfiguration` implementation. It has three jobs: window config, DI/page registration, and naming the root page.

```csharp
public class App : IAppConfiguration
{
    public void ConfigureWindow(PlusUiConfiguration configuration)
    {
        configuration.Title = "PlusUi Demo";
        configuration.Size = new SizeI(1200, 800);
        configuration.EnableNavigationStack = true;   // required for GoBack/PopToRoot
        configuration.RememberWindowPosition = true;
    }

    public void ConfigureApp(IPlusUiAppBuilder builder)
    {
        builder.AddPage<MainPage>().WithViewModel<MainPageViewModel>();
        builder.AddPage<DetailPage>().WithViewModel<DetailPageViewModel>();
    }

    // Returns the first page shown. Resolved through DI.
    public UiPageElement GetRootPage(IServiceProvider serviceProvider)
        => serviceProvider.GetRequiredService<MainPage>();
}
```

| Member | Purpose |
|---|---|
| `ConfigureWindow(PlusUiConfiguration)` | Window size/title/icon + navigation & accessibility flags. |
| `ConfigureApp(IPlusUiAppBuilder)` | Register pages, view models, popups, fonts, styles into DI. |
| `GetRootPage(IServiceProvider)` | Resolve and return the initial page instance. |

---

## Registering pages & view models

Registration is just DI registration via builder extension methods on `IPlusUiAppBuilder`. `AddPage`, `WithViewModel`, and `AddPopup` register as **transient**. `RegisterFont` and `StylePlusUi` register as **singleton**.

| Method | Effect | Constraint |
|---|---|---|
| `AddPage<TPage>()` | `services.AddTransient<TPage>()` | `TPage : UiPageElement` |
| `WithViewModel<TVM>()` | `services.AddTransient<TVM>()` | `TVM : class, INotifyPropertyChanged` |
| `AddPopup<TPopup>()` | `services.AddTransient<TPopup>()` | `TPopup : UiPopupElement` |
| `RegisterFont(resourcePath, fontFamily, fontWeight, fontStyle)` | Registers a font for the renderer | — |
| `StylePlusUi<TStyle>()` | Registers app-wide `IApplicationStyle` | `TStyle : IApplicationStyle` |

```csharp
builder.AddPage<MainPage>().WithViewModel<MainPageViewModel>();
```

- `AddPage` and `WithViewModel` both return the **same builder** — chaining is cosmetic. `WithViewModel` does **not** bind the VM to the preceding page; it only registers the VM type. The page receives its VM through its own constructor (see below).
- A VM shared by many pages is registered once: `builder.WithViewModel<DemoPageViewModel>();` — every page whose constructor takes `DemoPageViewModel` then resolves the same registration.
- Forgetting to register a page throws at navigation time: `InvalidOperationException` ("could not be resolved ... Ensure the page is registered").

---

## The root page & startup

The framework calls `PlusUiNavigationService.Initialize()` at startup:

1. Resolves `NavigationContainer`, transition/overlay services, and `PlusUiConfiguration` from DI.
2. Calls `IAppConfiguration.GetRootPage(serviceProvider)` to get the first page instance.
3. Pushes it as an **init call** (`isInitCall: true`): builds the page and calls `OnNavigatedTo(null)`, but skips `Disappearing`/`OnNavigatedFrom` on any previous page (there is none).

You never call `Initialize()` yourself.

---

## INavigationService API

Inject `INavigationService` into a ViewModel (constructor). It is the only navigation entry point.

| Member | Signature | Behavior |
|---|---|---|
| Navigate | `void NavigateTo<TPage>(object? parameter = null)` where `TPage : UiPageElement` | Resolve `TPage` from DI, push it, call its `OnNavigatedTo(parameter)`. |
| Back | `void GoBack()` | Pop current page, return to previous. Requires `EnableNavigationStack = true`. |
| To root | `void PopToRoot()` | Pop everything except the root page. Requires `EnableNavigationStack = true`. |
| Query | `bool CanGoBack { get; }` | `true` only when stack enabled **and** depth > 1. |
| Query | `int StackDepth { get; }` | Current stack depth (1 at root). |

```csharp
public partial class MainPageViewModel : ObservableObject
{
    private readonly INavigationService _navigation;
    public MainPageViewModel(INavigationService navigation) => _navigation = navigation;

    [RelayCommand] private void OpenDetail() => _navigation.NavigateTo<DetailPage>();
    [RelayCommand] private void OpenUser()   => _navigation.NavigateTo<DetailPage>(42);
}
```

Real Demo back command (`DemoPageViewModel`):

```csharp
public partial class DemoPageViewModel(INavigationService navigation) : ObservableObject
{
    [RelayCommand] private void GoBack() => navigation.GoBack();
}
```

Rules:
- `GoBack()` / `PopToRoot()` throw `InvalidOperationException` if `EnableNavigationStack = false`.
- `GoBack()` throws if already at the root (`CanGoBack == false`). Guard with `CanGoBack` or bind the back button only on non-root pages.
- `PopToRoot()` is a no-op (logs, returns) when already at root — it does **not** throw for that case.
- Navigating to the page type you are already on is **ignored** (early return after a debug log). You cannot "reload" the current page or replace its parameter via `NavigateTo<SamePage>(...)`.

---

## Navigating with parameters

`parameter` is an `object?` passed straight through to the target page's `OnNavigatedTo`. There is no type safety — cast in the override.

```csharp
// Caller
_navigation.NavigateTo<DetailPage>(new DetailArgs(userId: 42, edit: true));

// Target page
public class DetailPage(DetailPageViewModel vm) : UiPageElement(vm)
{
    public override void OnNavigatedTo(object? parameter)
    {
        if (parameter is DetailArgs args)
            vm.Load(args.UserId, args.Edit);
    }
    protected override UiElement Build() => /* ... */;
}
```

- The parameter is stored in the page's `NavigationStackItem` and re-passed to `OnNavigatedTo` when you `GoBack()` to that page.
- Prefer a single record/DTO over multiple loosely-typed values.

---

## Page <-> ViewModel relationship

A page is a class deriving from `UiPageElement`. Its VM is supplied via the **base constructor**, which requires an `INotifyPropertyChanged`:

```csharp
public class MainPage(MainPageViewModel vm) : UiPageElement(vm)
{
    protected override UiElement Build()
    {
        return new Grid()
            .AddColumn(Column.Auto).AddColumn(Column.Star).AddRow(Row.Star)
            .AddChild(BuildSidebar(), row: 0, column: 0)
            .AddChild(BuildWelcome(), row: 0, column: 1);
    }
}
```

- The primary-constructor parameter `vm` is captured and passed to `base(vm)`, **and** is directly usable inside `Build()` (e.g. `.BindItemsSource(() => vm.Rows)`, `.SetCommand(vm.NavigateToDemoCommand)`).
- `UiPageElement` exposes the VM as `ViewModel` (typed `INotifyPropertyChanged`). The framework subscribes to `ViewModel.PropertyChanged` while the page is current and calls `UpdateBindings(propertyName)` on the tree, so `Bind*` fluent methods refresh automatically. It unsubscribes on navigate-away.
- VMs use CommunityToolkit.Mvvm: `ObservableObject` + `[ObservableProperty]` / `[RelayCommand]`.
- `Build()` is abstract — every page implements it and returns the root `UiElement`. It runs inside `BuildPage()` (called by the navigation service), not in the constructor.
- Optional `protected override void ConfigurePageStyles(Style pageStyle)` applies page-scoped styles.

---

## Page lifecycle

Override the virtual hooks on `UiPageElement` as needed. Actual call order in `PlusUiNavigationService`:

**Forward navigation (`NavigateTo`)** — old page O, new page N:

| Step | Call | On |
|---|---|---|
| 1 | `Disappearing()` | O |
| 2 | `OnNavigatedFrom()` | O |
| 3 | unsubscribe `PropertyChanged` | O |
| 4 | `BuildPage()` -> `Build()` then `Appearing()` | N |
| 5 | subscribe `PropertyChanged` | N |
| 6 | `OnNavigatedTo(parameter)` | N |

**GoBack** — outgoing O, restored previous P:

| Step | Call | On |
|---|---|---|
| 1 | `Disappearing()` then `OnNavigatedFrom()` | O |
| 2 | unsubscribe | O |
| 3 | `UpdateBindings()` (state preserved) or `BuildPage()` | P |
| 4 | subscribe | P |
| 5 | `OnNavigatedTo(savedParameter)` then `Appearing()` | P |

| Hook | When |
|---|---|
| `Build()` | Build the UI tree (abstract). Runs during `BuildPage()`. |
| `Appearing()` | Page is about to be shown. On forward nav fires **inside** `BuildPage` (before `OnNavigatedTo`); on `GoBack` fires **after** `OnNavigatedTo`. |
| `OnNavigatedTo(object?)` | Receives the navigation parameter. Use for parameter-driven init. |
| `OnNavigatedFrom()` | Navigating away. Cancel work / save state. |
| `Disappearing()` | Page about to leave the screen. |
| `Dispose(bool)` | Disposes the page tree. Called when a page is popped/replaced and state is not preserved. |

Notes:
- Forward nav order is `Appearing` then `OnNavigatedTo`; back nav order is `OnNavigatedTo` then `Appearing`. Do not rely on a single fixed ordering across the two — put parameter handling in `OnNavigatedTo`, not `Appearing`.
- All overlays are dismissed (`IOverlayService.DismissAll()`) before every navigation.

---

## Navigation configuration (PlusUiConfiguration)

Set in `ConfigureWindow`. Navigation-relevant fields:

| Property | Default | Effect |
|---|---|---|
| `EnableNavigationStack` | `false` | When `false`, `NavigateTo` **replaces** the current page (old page disposed); `GoBack`/`PopToRoot` throw. When `true`, pages are pushed onto a stack. |
| `PreservePageState` | `true` | (stack on) Keep popped/background pages in memory and reuse on `GoBack` (bindings refreshed via `UpdateBindings`). When `false`, pages are disposed on leave and rebuilt. |
| `MaxStackDepth` | `50` | (stack on) `Push` throws `InvalidOperationException` when exceeded. |
| `DefaultTransition` | `new SlideTransition()` | Global page transition. Set to `new NoneTransition()` to disable. |
| `RespectReducedMotion` | `false` | Skip transitions when the OS requests reduced motion. |

Per-page transition override (wins over `DefaultTransition`):

```csharp
public class DetailPage : UiPageElement
{
    private readonly DetailPageViewModel _vm;
    public DetailPage(DetailPageViewModel vm) : base(vm)
    {
        _vm = vm;
        SetTransition(new NoneTransition());   // overrides DefaultTransition for this page
    }

    protected override UiElement Build() => /* ... */;
}
```

`SetTransition(IPageTransition?)` / the `Transition` property set a page-specific transition; `GoBack` automatically plays its reversed form.

---

## Navigation stack internals

`NavigationContainer` (singleton, DI) owns a `Stack<NavigationStackItem>`. Each `NavigationStackItem` holds `{ UiPageElement Page, object? Parameter }`.

| Member | Meaning |
|---|---|
| `CurrentPage` | Top of stack (throws if empty). |
| `CurrentParameter` | Parameter of the top page. |
| `CanGoBack` | `Count > 1`. |
| `StackDepth` | `Count`. |
| `IsStackEnabled` / `PreservePageState` | Mirror the configuration flags. |
| `Push(page, parameter)` | Push (stack on) or replace (stack off, disposing old). |
| `Pop()` | Pop top, return previous item; disposes popped page unless `PreservePageState`. |
| `PopToRoot()` | Drop all but root. |
| `PeekPrevious()` | Previous page without popping (or null). |

- Stack **disabled**: `Push` disposes the current page and replaces it — there is exactly one page; `StackDepth == 1`, `CanGoBack == false`.
- Stack enabled + `PreservePageState == false`: on `Push`, all background pages except the immediate current one are disposed (only one level of back is retained).

---

## PageChanged event

`NavigationContainer.PageChanged` (`EventHandler<PageChangedEventArgs>`) fires on push, `Pop`, and `PopToRoot`. `PageChangedEventArgs` exposes `NewPage` and `PreviousPage` (`PreviousPage` is `null` for the initial page). The host renderer uses this to swap the displayed page; app code rarely subscribes.

---

## Real example: master/detail flow

```csharp
// Registration (App.ConfigureApp)
builder.AddPage<ListPage>().WithViewModel<ListPageViewModel>();
builder.AddPage<DetailPage>().WithViewModel<DetailPageViewModel>();

// List VM -> navigate with id
public partial class ListPageViewModel(INavigationService nav) : ObservableObject
{
    [RelayCommand] private void Open(int id) => nav.NavigateTo<DetailPage>(id);
}

// Detail page -> read parameter, drive VM
public class DetailPage(DetailPageViewModel vm) : UiPageElement(vm)
{
    public override void OnNavigatedTo(object? parameter)
    {
        if (parameter is int id) vm.Load(id);
    }

    protected override UiElement Build() =>
        new VStack()
            .AddChild(new Label().BindText(() => vm.Title))
            .AddChild(new Button().SetText("Back").SetCommand(vm.BackCommand));
}

public partial class DetailPageViewModel(INavigationService nav) : ObservableObject
{
    [ObservableProperty] private string title = "";
    public void Load(int id) => Title = $"Item {id}";
    [RelayCommand] private void Back() => nav.GoBack();
}
```

---

## Common LLM mistakes

- **Calling `GoBack()`/`PopToRoot()` with the stack disabled.** They throw unless `EnableNavigationStack = true`. The Demo app sets it in `ConfigureWindow`.
- **Calling `GoBack()` at the root.** Throws `InvalidOperationException`. Check `CanGoBack` first, or only show a back button on non-root pages.
- **Expecting `NavigateTo<SamePage>(newParam)` to reload.** Navigating to the type you are already on is ignored. Pop/push or restructure instead.
- **Putting parameter handling in `Appearing()`.** Parameters arrive in `OnNavigatedTo(object?)`. Ordering of `Appearing` vs `OnNavigatedTo` differs between forward and back navigation.
- **Forgetting to register the page.** `NavigateTo<T>` resolves `T` via `GetRequiredService` and throws if unregistered. Add `builder.AddPage<T>()`.
- **Thinking `WithViewModel` binds a VM to the preceding `AddPage`.** It only registers the VM type in DI. The page gets its VM through its own constructor parameter passed to `base(vm)`.
- **Doing work in the page constructor instead of `Build()`.** The tree is built in `BuildPage()` (invoked by the navigation service), not at construction. The constructor should just take the VM and call `base(vm)`.
- **Typed-parameter assumptions.** The parameter is `object?`; always pattern-match/cast (`if (parameter is T t)`), never assume a type.
- **Forgetting transitions reverse on back.** A custom page `Transition` is auto-reversed by `GoBack`; do not add a second "back" transition.
