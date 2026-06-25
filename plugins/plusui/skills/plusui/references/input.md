# PlusUi Input Controls

Interactive controls that capture user input or trigger actions: `Button`, `Checkbox`, `RadioButton`, `Toggle`, `Slider`, `ComboBox<T>`, `DatePicker`, `TimePicker`. All are built fluently in C# (no XAML) and follow the project-wide rule that every `Set*` method has a matching `Bind*` method. Editable controls add a **two-way** `Bind*` overload that also takes a setter `Action<T>` to write user edits back to the ViewModel. For binding mechanics see [binding-and-state.md](binding-and-state.md).

## Table of contents

- [Conventions](#conventions)
- [Button](#button)
- [Checkbox](#checkbox)
- [RadioButton](#radiobutton)
- [Toggle](#toggle)
- [Slider](#slider)
- [ComboBox&lt;T&gt;](#comboboxt)
- [DatePicker](#datepicker)
- [TimePicker](#timepicker)
- [Common LLM mistakes](#common-llm-mistakes)

## Conventions

- **Bind?** column: `yes` = a `Bind*` overload exists (one-way `() => vm.X`, and two-way `() => vm.X, v => vm.X = v` for editable values). `no` = `Set*`/callback only, no live binding.
- Inputs whose value the user changes (`IsChecked`, `IsOn`, `Value`, `SelectedItem`, `SelectedDate`, `SelectedTime`, `Text` on `Entry`) need the **two-way** overload with a setter, or edits are discarded.
- `Color`/`new Color(r,g,b[,a])` and `Colors.*` are used by most controls. `ComboBox<T>`, `DatePicker`, and `TimePicker` use SkiaSharp's `SKColor` for their text/popup color properties (`new SKColor(r,g,b)` / `SKColors.*`), not `Color`.
- Pass **lambda expressions** to `Bind*`, never `nameof(...)` strings. The `nameof(...)` form is obsolete and will not compile against the current API.

## Button

A clickable control showing text and/or an icon. Runs `OnClick` then, if present and `CanExecute` is true, its `ICommand`. Has a built-in hover background. Base class `UiTextElement`. Centers its text by default (unlike `Label`).

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `Text` | `string` | `null` | yes | Button label. |
| `Command` | `ICommand` | `null` | yes | Executed on click after `OnClick`. |
| `CommandParameter` | `object` | `null` | yes | Passed to `CanExecute`/`Execute`. |
| `OnClick` | `Action` | `null` | yes | Fires before `Command`. |
| `Icon` | `string` | `null` | yes | Path to image/SVG. |
| `IconPosition` | `IconPosition` | `PlusUiDefaults.IconPosition` | yes | Flags: `None`, `Leading`, `Trailing`, or both. |
| `IconTintColor` | `Color` | `null` | yes | SVG only. |
| `HoverBackground` | `IBackground` | `BackgroundHover` | yes | Drawn over `Background` while hovered. |
| `Padding` | `Margin` | `Margin(0)` | yes | Internal spacing around content. |
| `TextSize` | `float` | `12` | yes | Icon size matches this. |
| `TextColor` | `Color` | `TextPrimary` | yes | |
| `FontWeight` | `FontWeight` | default | yes | |
| `FontStyle` | `FontStyle` | default | yes | |
| `Background` | `IBackground` | `BackgroundControl` | yes | |
| `CornerRadius` | `float` | default | yes | |
| `Margin` | `Margin` | `Margin(0)` | yes | External spacing. |
| `HorizontalAlignment` | `HorizontalAlignment` | `Undefined` | yes | `Stretch` for full width. |
| `IsVisible` | `bool` | `true` | yes | |
| `TabIndex` | `int?` | `null` | yes | |
| `TabStop` | `bool` | `true` | yes | |
| `AccessibilityLabel` | `string` | `null` | yes | Defaults to `Text`. |

```csharp
// Command binding (most common)
new Button()
    .SetText("Click Me")
    .SetCommand(vm.ClickCommand);

// Icon + text, leading icon, tinted SVG
new Button()
    .SetIcon("Assets/Icons/save.svg")
    .SetIconTintColor(Colors.White)
    .SetText("Save")
    .SetIconPosition(IconPosition.Leading)
    .SetTextColor(Colors.White)
    .SetBackground(new Color(52, 199, 89))
    .SetPadding(new Margin(12, 8))
    .SetOnClick(() => SaveDocument());

// Icon-only ghost button with custom hover
new Button()
    .SetIcon("Assets/Icons/settings.svg")
    .SetIconTintColor(Colors.Gray)
    .SetBackground(Colors.Transparent)
    .SetHoverBackground(new Color(128, 128, 128, 50))
    .SetCornerRadius(4)
    .SetPadding(new Margin(8))
    .SetOnClick(() => OpenSettings());
```

**Gotchas**
- `OnClick` and `Command` both run (OnClick first). Use `OnClick` alone when you don't need `ICommand` infrastructure.
- Icon size is tied to `TextSize` — you cannot enlarge the icon independently; ship a pre-sized SVG instead.
- `IconTintColor` works only on SVGs; PNG/JPG ignore it.
- A default hover background is preconfigured. Call `.SetHoverBackground(null)` to disable it.
- Text is centered by default; set horizontal text alignment explicitly for left/right.
- No built-in disabled state — rely on `ICommand.CanExecute` (a `RelayCommand` predicate disables `Execute` automatically) or toggle visibility/opacity.
- Use `IconPosition.Leading | IconPosition.Trailing` to show icons on both sides.

## Checkbox

A binary on/off box with a checkmark. No built-in label — pair with a `Label` in an `HStack`. Base `UiElement` (`IToggleButtonControl`, `IFocusable`). Default size 24x24.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `IsChecked` | `bool` | `false` | yes (two-way) | Internal setter — change via `SetIsChecked`/`Toggle`/binding only. |
| `Color` | `Color` | `TextPrimary` | yes | Stroke + checkmark only, not the box fill. |
| `Background` | `IBackground`/`Color` | — | yes | Box fill. |
| `DesiredSize` | `Size` | `24x24` | yes | |
| `Margin` | `Margin` | `0` | yes | |
| `HorizontalAlignment` | `HorizontalAlignment` | — | yes | |
| `VerticalAlignment` | `VerticalAlignment` | — | yes | |
| `IsVisible` | `bool` | `true` | yes | |
| `Opacity` | `float` | `1.0` | yes | |
| `TabIndex` | `int?` | `null` | yes | |
| `TabStop` | `bool` | `true` | yes | |
| `AccessibilityLabel` | `string?` | `"Checkbox"` | yes | |
| `AccessibilityValue` | `string?` | auto | yes | Auto `"Checked"`/`"Unchecked"`. |

Callback (no binding): `SetOnIsCheckedChanged(Action<bool>)`.

```csharp
// Labeled, two-way bound
new HStack()
    .SetSpacing(8)
    .AddChild(new Checkbox().BindIsChecked(() => vm.Accepted, v => vm.Accepted = v))
    .AddChild(new Label().BindText(() => vm.AcceptedText));

// Styled
new Checkbox()
    .SetColor(new Color(0, 122, 255))
    .SetDesiredSize(new Size(28, 28))
    .SetBackground(new Color(40, 40, 40));

// Accessible
new Checkbox()
    .SetAccessibilityLabel("Accept terms and conditions")
    .SetAccessibilityHint("Check to accept the terms")
    .BindIsChecked(() => vm.AcceptTerms, v => vm.AcceptTerms = v);
```

**Gotchas**
- `IsChecked` is internal — never assign it directly; use `SetIsChecked`, `Toggle`, or binding.
- `SetColor` styles only the stroke/checkmark; use `SetBackground` for the box fill.
- `Toggle()` fires callbacks/bindings; `SetIsChecked()` does **not** fire `SetOnIsCheckedChanged`.
- No built-in label text — always pair with a `Label`.
- Generic `BindIsChecked<T>(getter, setter, toControl, toSource)` exists for non-bool VM properties.

## RadioButton

Mutually exclusive selection within a `Group`. Setting one to selected deselects others sharing the same group. Includes its own label `Text`. Base `UiElement` (`IInputControl`, `IFocusable`). Circle size fixed (20px).

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `IsSelected` | `bool` | `false` | yes (two-way, setter optional) | Setting `true` deselects others in the group. |
| `Group` | `object` | `null` | yes | Group key (string/enum/object). **Required** for exclusivity. |
| `Value` | `object` | `null` | yes | Program identifier; distinct from `Text`. |
| `Text` | `string` | `null` | yes | Display label. |
| `TextSize` | `float` | `14` | yes | |
| `TextColor` | `Color` | `TextPrimary` | yes | |
| `CircleColor` | `Color` | `TextPrimary` | yes | Outline when unselected. |
| `SelectedColor` | `Color` | `AccentSuccess` | yes | Outline + fill when selected. |

```csharp
new VStack()
    .SetSpacing(8)
    .AddChild(new RadioButton().SetText("Red").SetGroup("color").SetValue("R").SetIsSelected(true))
    .AddChild(new RadioButton().SetText("Green").SetGroup("color").SetValue("G"))
    .AddChild(new RadioButton().SetText("Blue").SetGroup("color").SetValue("B"));

// Two-way bound to a VM flag
new RadioButton()
    .SetText("Express shipping")
    .SetGroup("shipping")
    .SetValue("express")
    .BindIsSelected(() => vm.IsExpress, v => vm.IsExpress = v);
```

**Gotchas**
- `Group` MUST be set or buttons act independently (all appear separately selectable).
- `Text` (visible label) and `Value` (program identifier) are distinct — e.g. `Text="Large"`, `Value="L"`.
- Buttons in **different** groups can each be selected — by design, not a bug.
- Set `Group`/`Value` in the initial fluent chain; changing `Group` later forces re-registration with the `RadioButtonManager`.
- The `RadioButtonManager` is resolved from DI; if not initialized, grouping silently fails (no error).
- Circle size is fixed (`CheckboxSize` = 20px) and not customizable.

## Toggle

iOS/Android style on/off switch with a sliding thumb. No built-in label — pair with a `Label`. Base `UiElement` (`IToggleButtonControl`, `IFocusable`). Default size 50x28.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `IsOn` | `bool` | `false` | yes (two-way) | Internal setter — use `SetIsOn`/`BindIsOn` only. |
| `OnColor` | `Color` | `AccentSuccess` | yes | Track when on. |
| `OffColor` | `Color` | `new Color(120,120,128)` | yes | Track when off. |
| `ThumbColor` | `Color` | `TextPrimary` | yes | Independent of state. |
| `DesiredSize` | `Size` | `50x28` | yes | Raise to >=44x44 for touch. |
| `Margin` | `Margin` | `0` | yes | |
| `HorizontalAlignment` | `HorizontalAlignment` | `Undefined` | yes | |
| `VerticalAlignment` | `VerticalAlignment` | `Undefined` | yes | |
| `Background` | `IBackground`/`Color` | `null` | yes | Rarely set. |
| `IsVisible` | `bool` | `true` | yes | |
| `Opacity` | `float` | `1.0` | yes | |
| `TabIndex` | `int?` | `null` | yes | |
| `TabStop` | `bool` | `true` | yes | |
| `AccessibilityLabel` | `string?` | `"Toggle switch"` | yes | |
| `AccessibilityHint` | `string?` | `null` | yes | |

```csharp
// Initial state only
new Toggle().SetIsOn(true);

// Two-way + custom colors
new Toggle()
    .BindIsOn(() => vm.IsDarkMode, v => vm.IsDarkMode = v)
    .SetOnColor(Colors.Blue)
    .SetOffColor(Colors.Gray);

// Settings row
new HStack()
    .SetVerticalAlignment(VerticalAlignment.Center)
    .AddChildren(
        new Label().SetText("Dark Mode"),
        new Toggle()
            .SetMargin(new Margin(12, 0, 0, 0))
            .BindIsOn(() => vm.Enabled, v => vm.Enabled = v));
```

**Gotchas**
- `IsOn` is internal — use `SetIsOn`/`BindIsOn`, never the property directly.
- `BindIsOn(getter, setter)` takes `Expression<Func<bool>>` + `Action<bool>`; omit the setter only for one-way.
- Generic `BindIsOn<T>(getter, setter, toControl, toSource)` exists for non-bool VM properties.
- Default 50x28 is below the 44x44 touch minimum — call `SetDesiredSize` for accessibility.
- No animation; only the thumb position changes. No tri-state/indeterminate.
- `ThumbColor` does not change with state — drive it from a computed property if needed.
- Accessibility value auto-computes to `"On"`/`"Off"`.

## Slider

Pick a numeric value in a range by dragging the thumb or via keyboard. Default range 0-100, height 30px, stretches horizontally. Base `UiElement`. Only the thumb is interactive (clicking the track does nothing).

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `Value` | `float` | `0` | yes (two-way) | Auto-clamped to `[Minimum, Maximum]`. |
| `Minimum` | `float` | `0` | yes | Raising it clamps `Value` up. |
| `Maximum` | `float` | `100` | yes | Lowering it clamps `Value` down. |
| `MinimumTrackColor` | `Color` | `AccentPrimary` | yes | Filled portion. |
| `MaximumTrackColor` | `Color` | `TrackColor` | yes | Unfilled portion. |
| `ThumbColor` | `Color` | `TextPrimary` | yes | |
| `Margin` | `Margin` | none | yes | |
| `HorizontalAlignment` | `HorizontalAlignment` | `Stretch` | yes | |
| `Background` | `IBackground`/`Color` | none | yes | |

Callback (no binding): `SetOnValueChanged(Action<float>)`.

```csharp
// Fixed range, initial value
new Slider()
    .SetMinimum(0)
    .SetMaximum(10)
    .SetValue(5);

// Two-way bound + custom track colors
new Slider()
    .BindValue(() => vm.Value, v => vm.Value = v)
    .SetMinimum(0)
    .SetMaximum(100)
    .SetMinimumTrackColor(new Color(52, 199, 89))
    .SetMaximumTrackColor(new Color(60, 60, 60));

// Negative range with callback
new Slider()
    .SetMinimum(-20)
    .SetMaximum(50)
    .SetOnValueChanged(value => Console.WriteLine($"Value: {value}"));
```

**Gotchas**
- `BindValue` takes `Expression<Func<float>>`; the generic `BindValue<T>(getter, setter, toControl, toSource)` handles `int`/`decimal` VM properties.
- Keyboard arrows move by **5% of the range** (0-10 → 0.5; 0-100 → 5), not fixed steps.
- Auto-clamps from `Minimum`/`Maximum` changes fire no separate value-change event.
- Constrain width with `SetHorizontalAlignment(HorizontalAlignment.Left)` + `SetDesiredWidth(...)`; height via `SetDesiredHeight(...)`.
- Focusable by default; `SetTabStop(false)` to remove from tab order.

## ComboBox&lt;T&gt;

Dropdown single-select over `ItemsSource`. Generic — always specify `T`. Base `UiElement` (`IInputControl`, `IFocusable`, `IKeyboardInputHandler`). Default size 200x40. Text/popup colors use `SKColor`.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `ItemsSource` | `IEnumerable<T>?` | `null` | yes | Supports `INotifyCollectionChanged`. |
| `SelectedItem` | `T?` | `default(T)` | yes (two-way) | Synced with `SelectedIndex`. |
| `SelectedIndex` | `int` | `-1` | yes (two-way) | `-1` = nothing selected. |
| `DisplayFunc` | `Func<T,string>` | `ToString()` | yes | How items render. |
| `Placeholder` | `string?` | `null` | yes | Shown when nothing selected. |
| `PlaceholderColor` | `SKColor` | `TextPlaceholder` | yes | |
| `TextColor` | `SKColor` | `TextPrimary` | yes | |
| `TextSize` | `float` | `14` | yes | |
| `FontFamily` | `string?` | `null` | yes | |
| `DropdownBackground` | `SKColor` | `BackgroundPrimary` | yes | |
| `HoverBackground` | `SKColor` | `BackgroundHover` | yes | |
| `IsOpen` | `bool` | `false` | yes | Programmatically open/close. |
| `OnSelectionChanged` | `Action<T?>` | `null` | **no** | `Set` only — use `BindSelectedItem` for VM sync. |
| `Padding` | `Margin` | `12h, 8v` | yes | |
| `Margin` | `Margin` | `0` | yes | |
| `Background` | `IBackground` | `BackgroundInput` | yes | |
| `CornerRadius` | `float` | default | yes | |
| `HorizontalAlignment` | `HorizontalAlignment` | `Left` | yes | `Stretch` for full width. |
| `VerticalAlignment` | `VerticalAlignment` | `Top` | yes | |
| `TabIndex` | `int?` | `null` | yes | |
| `AccessibilityLabel` | `string?` | `null` | yes | Falls back to `Placeholder`/`"Dropdown"`. |

```csharp
// Simple string list with callback
new ComboBox<string>()
    .SetItemsSource(new[] { "Red", "Green", "Blue", "Yellow" })
    .SetPlaceholder("Choose a color...")
    .SetOnSelectionChanged(v => Console.WriteLine(v));

// Custom objects, two-way bound, custom display
new ComboBox<Country>()
    .BindItemsSource(() => vm.Countries)
    .SetDisplayFunc(c => $"{c.Flag} {c.Name}")
    .BindSelectedItem(() => vm.SelectedCountry, c => vm.SelectedCountry = c)
    .SetDesiredWidth(250);

// Enum source, styled dropdown
new ComboBox<Priority>()
    .SetItemsSource(Enum.GetValues<Priority>())
    .SetSelectedItem(Priority.Normal)
    .SetBackground(new Color(50, 50, 50))
    .SetCornerRadius(8)
    .SetDropdownBackground(new SKColor(40, 40, 40))
    .SetHoverBackground(new SKColor(70, 70, 70));
```

**Gotchas**
- For custom objects always set `DisplayFunc`, or you get the type name from `ToString()`.
- `OnSelectionChanged` has no `Bind` version — for VM sync use `BindSelectedItem(getter, setter)`.
- `SelectedItem`/`SelectedIndex` stay synced; if the collection shrinks past the selection, index becomes `-1` and item becomes `default(T)`.
- Full width: `SetHorizontalAlignment(HorizontalAlignment.Stretch)`. Default is 200x40.
- Use an `ObservableCollection<T>` + `BindItemsSource` for live add/remove; no manual `SetItemsSource` needed.
- Dropdown max height 200px (scrolls internally) and auto-positions upward when space below is tight.
- Keyboard: when closed Enter/Space/arrows open; when open Escape closes, Enter/Space selects, arrows navigate.

## DatePicker

Calendar-popup picker for a single `DateOnly?`. Base `UiElement`. Default size 200x40. Colors use `SKColor`. **No default background** — renders as floating text with an icon until you set one.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `SelectedDate` | `DateOnly?` | `null` | yes (two-way) | |
| `MinDate` | `DateOnly?` | `null` | yes | Earlier dates rejected/greyed. |
| `MaxDate` | `DateOnly?` | `null` | yes | Later dates rejected/greyed. |
| `DisplayFormat` | `string` | `"dd.MM.yyyy"` | yes | .NET date format. |
| `TextColor` | `SKColor` | `TextPrimary` | yes | |
| `TextSize` | `float` | `FontSize` | yes | |
| `FontFamily` | `string?` | `null` | yes | |
| `Placeholder` | `string?` | `null` | yes | |
| `PlaceholderColor` | `SKColor` | `TextPlaceholder` | yes | |
| `Padding` | `Margin` | `Margin(12, 8)` | yes | |
| `WeekStart` | `DayOfWeekStart` | `Monday` | yes | `Monday`/`Sunday`. |
| `ShowTodayButton` | `bool` | `true` | yes | |
| `IsOpen` | `bool` | `false` | yes | |
| `CalendarBackground` | `SKColor` | `BackgroundPrimary` | yes | |
| `HoverBackground` | `SKColor` | `BackgroundHover` | yes | |
| `SelectedBackground` | `SKColor` | `BackgroundSelected` | yes | |
| `TodayBorderColor` | `SKColor` | `AccentPrimary` | yes | Drawn only when today is not selected. |
| `OnSelectedDateChanged` | `Action<DateOnly?>` | `null` | **no** | `Set` only. |

```csharp
// Today preselected
new DatePicker()
    .SetPlaceholder("Select date...")
    .SetDisplayFormat("dd.MM.yyyy")
    .SetSelectedDate(DateOnly.FromDateTime(DateTime.Today));

// Date of birth, range-limited, two-way bound, visible background
new DatePicker()
    .SetPlaceholder("Date of birth")
    .SetDisplayFormat("dd MMMM yyyy")
    .SetMinDate(new DateOnly(1900, 1, 1))
    .SetMaxDate(DateOnly.FromDateTime(DateTime.Today))
    .SetBackground(new Color(35, 35, 35))
    .BindSelectedDate(() => vm.BirthDate, d => vm.BirthDate = d);

// Sunday-first, styled calendar
new DatePicker()
    .SetDisplayFormat("MM/dd/yyyy")
    .SetWeekStart(DayOfWeekStart.Sunday)
    .SetCalendarBackground(new SKColor(35, 35, 35))
    .SetSelectedBackground(new SKColor(66, 165, 245))
    .BindSelectedDate(() => vm.Date, d => vm.Date = d);
```

**Gotchas**
- No default background/border — add `.SetBackground(...)` to look like an input field.
- Two-way needs the getter+setter overload: `BindSelectedDate(() => vm.Date, d => vm.Date = d)`. The single-getter form is one-way. (`nameof`-prefixed overloads in old docs are obsolete.)
- Keyboard entry tries formats in order: `dd.MM.yyyy`, `d.M.yyyy`, `dd/MM/yyyy`, `d/M/yyyy`, `yyyy-MM-dd`, `ddMMyyyy` — ambiguous input (e.g. `12/13/2024`) may parse wrong.
- Losing focus closes the popup. `MinDate`/`MaxDate` out-of-range selections are rejected silently.
- Calendar nav: arrows (±1 day / ±7 days), Enter/Space select, Escape closes; click/Enter/Space or `IsOpen=true` opens.

## TimePicker

Hour/minute popup picker for a single `TimeOnly?`, 12/24-hour. Base `UiElement` (`IInputControl`, `ITextInputControl`, `IHoverableControl`, `IFocusable`, `IKeyboardInputHandler`). Default size 150x40. Colors use `SKColor`. **No default background**.

| Property | Type | Default | Bind? | Notes |
|----------|------|---------|-------|-------|
| `SelectedTime` | `TimeOnly?` | `null` | yes (two-way) | Rounds to `MinuteIncrement`. |
| `MinTime` | `TimeOnly?` | `null` | yes | Inclusive. |
| `MaxTime` | `TimeOnly?` | `null` | yes | Inclusive. |
| `MinuteIncrement` | `int` | `1` | yes | Only 1/5/10/15/30; others silently become 1. |
| `Is24HourFormat` | `bool` | `true` | yes | `false` auto-sets `DisplayFormat` to `hh:mm tt`. |
| `DisplayFormat` | `string` | `"HH:mm"` | yes | .NET time format. |
| `TextColor` | `SKColor` | `TextPrimary` | yes | |
| `TextSize` | `float` | `FontSize` | yes | |
| `FontFamily` | `string?` | `null` | yes | |
| `SelectorBackground` | `SKColor` | `BackgroundPrimary` | yes | |
| `HoverBackground` | `SKColor` | `BackgroundHover` | yes | |
| `SelectedBackground` | `SKColor` | `BackgroundSelected` | yes | |
| `IsOpen` | `bool` | `false` | yes | |
| `Placeholder` | `string` | empty | yes | |
| `PlaceholderColor` | `SKColor` | `TextPlaceholder` | yes | |
| `Padding` | `Margin` | `12h, 8v` | yes | |
| `Margin` | `Margin` | `0` | yes | |
| `Background` | `IBackground`/`SKColor` | none | yes | |
| `CornerRadius` | `float` | `0` | yes | |

```csharp
// Quarter-hour increments, 24h, two-way bound
new TimePicker()
    .SetSelectedTime(new TimeOnly(9, 0))
    .SetMinuteIncrement(15)
    .Set24HourFormat(true)
    .BindSelectedTime(() => vm.StartTime, v => vm.StartTime = v);

// Range-limited meeting time with a styled button
new TimePicker()
    .SetPlaceholder("Meeting time")
    .SetMinuteIncrement(15)
    .SetMinTime(new TimeOnly(9, 0))
    .SetMaxTime(new TimeOnly(17, 0))
    .SetBackground(new SKColor(45, 45, 45))
    .SetCornerRadius(8)
    .BindSelectedTime(() => vm.MeetingTime, t => vm.MeetingTime = t);

// Interdependent start/end range
new HStack()
    .AddChildren(
        new TimePicker()
            .SetPlaceholder("Start")
            .BindSelectedTime(() => vm.StartTime, t => vm.StartTime = t)
            .BindMaxTime(() => vm.EndTime),
        new Label().SetText("to").SetMargin(new Margin(8, 0)),
        new TimePicker()
            .SetPlaceholder("End")
            .BindSelectedTime(() => vm.EndTime, t => vm.EndTime = t)
            .BindMinTime(() => vm.StartTime));
```

**Gotchas**
- `MinuteIncrement` validation is silent — only 1/5/10/15/30 are honored; anything else becomes 1.
- `SelectedTime` is nullable — check `HasValue` before reading `.Hour`/`.Minute`, or coalesce.
- Selector does **not** auto-close on selection; user must click outside, press Escape, or `SetIsOpen(false)`.
- Two-way needs `BindSelectedTime(getter, setter)`; the single-getter form is one-way.
- Set `DisplayFormat` before `Set24HourFormat`, or the auto-switch overrides your format.
- Colors here are SkiaSharp `SKColor` — use `new SKColor(r,g,b)`/`SKColors.*`, not `Colors.*`.
- No default background — add `.SetBackground(...)`/`.SetCornerRadius(...)` for a button look.
- `MinTime`/`MaxTime` block selection but do not grey out items in the popup.
- The popup needs `IOverlayService` from DI; if missing it silently fails to render.

## Common LLM mistakes

- **Passing `nameof(...)` to `Bind*`.** The current API is expression-only: `BindSelectedDate(() => vm.Date, d => vm.Date = d)`, not `BindSelectedDate(nameof(vm.Date), ...)`.
- **Omitting the setter on inputs.** One-arg `Bind*` is one-way; user edits to `Checkbox`/`Toggle`/`Slider`/`ComboBox`/`DatePicker`/`TimePicker` are discarded without `(getter, setter)`.
- **Assigning internal state properties directly.** `IsChecked`, `IsOn`, `RadioButton.Text`/`Value` have internal setters — use the `Set*`/`Bind*` fluent methods.
- **Using `Set*` for live data.** `SetText(vm.X)` captures once; use `BindText(() => vm.X)`.
- **Wrong color type.** `ComboBox<T>`/`DatePicker`/`TimePicker` color props are `SKColor`, not `Color`/`Colors.*`.
- **Expecting `Checkbox`/`Toggle`/`Slider` to show a label.** None render text — pair with a `Label` in an `HStack`/`VStack`.
- **Forgetting `Group` on `RadioButton`.** Without it, buttons don't coordinate selection.
- **Relying on `OnSelectionChanged`/`OnValueChanged`/`OnIsCheckedChanged` for VM sync.** These are fire-and-forget callbacks (no `Bind`); use the two-way `Bind*` to write back to a ViewModel.
- **`SetCommand(() => vm.Save())`.** `Command` is an `ICommand` (`vm.SaveCommand`); for a lambda use `SetOnClick(() => ...)`.
- **No visible field for `DatePicker`/`TimePicker`.** They have no default background — add `.SetBackground(...)`.
