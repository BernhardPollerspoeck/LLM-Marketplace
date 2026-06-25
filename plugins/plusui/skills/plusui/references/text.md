# PlusUi Text Controls

Controls for displaying and editing text: **Label** (static/dynamic text), **RichTextLabel** (mixed-style inline runs), **Link** (clickable hyperlink), and **Entry** (text input). UI is built in C# with the fluent builder — no XAML.

## Table of contents

- [Conventions (read first)](#conventions-read-first)
- [Label](#label)
- [RichTextLabel](#richtextlabel)
- [Link](#link)
- [Entry](#entry)
- [Common LLM mistakes](#common-llm-mistakes)

## Conventions (read first)

- **Every `SetX` has a matching `BindX`.** `Set*` writes a static value once; `Bind*` registers a live binding that re-runs on `PropertyChanged`. Property tables below only flag the rare property that has **no** `Bind*`.
- **`Bind*` takes a C# expression, never a property-name string.** Signature is `BindText(() => vm.Name)` — an `Expression<Func<T>>`. (The old `BindText(nameof(vm.Name), () => vm.Name)` form seen in some XML doc comments is NOT a real overload — do not use it.)
- **`Label`, `RichTextLabel`, `Link` are read-only bindings** (VM -> control). Only **`Entry`** has a two-way `BindText` overload that also takes a setter `Action<T>` to push edits back.
- The `[GenerateShadowMethods]` source generator makes inherited `Set*`/`Bind*` return the **derived** type, so chaining stays typed (e.g. `new Label().SetText(...)` returns `Label`).
- `Label`, `Link`, `Entry` extend `UiTextElement` and share its text properties (`Text`, `TextSize`, `TextColor`, `FontFamily`, `FontWeight`, `FontStyle`, `HorizontalTextAlignment`, `TextWrapping`, `MaxLines`, `TextTruncation`). `RichTextLabel` extends `UiElement` and re-declares its own text defaults that apply to child runs.
- `TextWrapping` values: `NoWrap` (default), `Wrap` (character break), `WordWrap` (word boundaries). **Wrapping needs an explicit width** (`SetDesiredWidth`) or it has no boundary to wrap against.
- `HorizontalTextAlignment` (Left/Center/Right) positions text *inside* the element; `HorizontalAlignment` (Left/Center/Right/Stretch) positions the *element* in its parent. They are independent.

---

## Label

Non-editable text for static or dynamic content, with wrapping, truncation, and binding. `IsFocusable => false`, `AccessibilityRole.Label`.

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Text` | `string?` | `null` | Content. Also feeds the accessibility label. |
| `TextSize` | `float` | `12` | Font size (points). |
| `TextColor` | `Color` | `White` | |
| `FontFamily` | `string` | theme | Font family name. |
| `FontWeight` | `FontWeight` | `Regular` | Thin/Light/Regular/SemiBold/Bold/Black… Defaults to Regular (400), not Bold. |
| `FontStyle` | `FontStyle` | `Normal` | Normal/Italic/Oblique. |
| `HorizontalTextAlignment` | `HorizontalTextAlignment` | `Left` | Text position within bounds. |
| `TextWrapping` | `TextWrapping` | `NoWrap` | Set to `Wrap`/`WordWrap` to enable wrapping. |
| `MaxLines` | `int?` | `null` | Caps visible lines; only honored when wrapping. |
| `TextTruncation` | `TextTruncation` | `None` | None/Start/Middle/End ellipsis. |
| `SupportsSystemFontScaling` | `bool` | global config | **No `Bind*`** — use `SetSupportsSystemFontScaling`. |
| `Margin` | `Margin` | zero | Outer spacing (inherited). |
| `HorizontalAlignment` / `VerticalAlignment` | enum | `Undefined` | Element placement in parent. |
| `DesiredWidth` / `DesiredHeight` | `float` | — | Explicit size. **Width is required for wrapping.** |
| `Background` | `IBackground` | `null` | Solid color or gradient. |
| `CornerRadius` | `float` | `0` | |
| `Opacity` | `float` | `1.0` | |
| `IsVisible` | `bool` | `true` | |
| `AccessibilityLabel` | `string?` | from `Text` | Screen-reader label. |
| `ShadowColor` / `ShadowOffset` / `ShadowBlur` | `Color` / `Point` / `float` | — | Text shadow. |

```csharp
// Static
new Label().SetText("The quick brown fox jumps over the lazy dog.");

// Styled
new Label()
    .SetText("Bold Accent Text")
    .SetFontWeight(FontWeight.Bold)
    .SetTextColor(PlusUiDefaults.AccentPrimary)
    .SetTextSize(18);

// Wrapping (needs both TextWrapping AND DesiredWidth)
new Label()
    .SetText("This long line wraps across multiple lines when WordWrap is on and width is limited.")
    .SetTextWrapping(TextWrapping.WordWrap)
    .SetDesiredWidth(360)
    .SetMaxLines(2)
    .SetTextTruncation(TextTruncation.End);

// One-way data binding (expression, not a string)
new Label().BindText(() => vm.UserName);
new Label().BindText(() => vm.Count, c => $"Items: {c}");   // generic overload with formatter
```

**Gotchas**
- `NoWrap` is the default — text never wraps unless you set `Wrap`/`WordWrap`.
- Wrapping silently does nothing without `SetDesiredWidth()`.
- Text is **clipped** to element bounds; oversized text does not overflow.
- `MaxLines` is ignored in `NoWrap` mode; truncation/ellipsis applies to the last visible line.
- In `NoWrap`, newlines and control characters are stripped/normalized to spaces; wrap modes honor explicit line breaks.
- Not focusable — inherited focus-ring properties exist but are inactive.

---

## RichTextLabel

Displays multiple styled segments (`TextRun`) inline within one control. Each run can override color, size, weight, style, and family; unset (`null`) run properties **inherit the label's defaults** (not theme defaults). `IsFocusable => false`, `AccessibilityRole.Label` (label is the concatenation of run text).

**RichTextLabel** properties (defaults supplied to runs that leave a value `null`):

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `TextSize` | `float` | `PlusUiDefaults.FontSize` | Default for runs with `FontSize == null`. |
| `TextColor` | `Color` | `PlusUiDefaults.TextPrimary` | Default for runs with `Color == null`. |
| `FontWeight` | `FontWeight` | `PlusUiDefaults.FontWeight` | |
| `FontStyle` | `FontStyle` | `PlusUiDefaults.FontStyle` | |
| `FontFamily` | `string?` | `null` | |
| `TextWrapping` | `TextWrapping` | theme | `NoWrap`, `Wrap` (character break), or `WordWrap` (word boundaries). |
| `HorizontalTextAlignment` | `HorizontalTextAlignment` | theme | |
| `MaxLines` | `int?` | `null` | Drops lines beyond the limit — **no ellipsis**. |
| `Margin` / `Background` / `DesiredWidth` / `DesiredHeight` / `IsVisible` / `Opacity` | inherited from `UiElement` | — | All have `Bind*`. |

Run management: `AddRun(TextRun)` appends; `SetRuns(IEnumerable<TextRun>)` **clears then adds**; `ClearRuns()`; `BindRuns(() => vm.Runs)` for reactive lists.

**TextRun** — construct with `new TextRun(text)`; style via fluent setters (each property is nullable to inherit):

| Method | Type | Bind |
|--------|------|------|
| `SetText` | `string` | `BindText(() => …)` (non-nullable `Func<string>`) |
| `SetColor` | `Color?` | `BindColor` |
| `SetFontSize` | `float?` | `BindFontSize` |
| `SetFontWeight` | `FontWeight?` | `BindFontWeight` |
| `SetFontStyle` | `FontStyle?` | `BindFontStyle` |
| `SetFontFamily` | `string?` | `BindFontFamily` |

```csharp
// Mixed colors / weights
new RichTextLabel()
    .AddRun(new TextRun("Hello "))
    .AddRun(new TextRun("World").SetColor(Colors.Blue).SetFontWeight(FontWeight.Bold));

// Price tag with per-run sizes (runs inherit TextSize=14 where not overridden)
new RichTextLabel()
    .SetTextSize(14)
    .SetTextWrapping(TextWrapping.WordWrap)
    .SetDesiredWidth(400)
    .AddRun(new TextRun("Price: "))
    .AddRun(new TextRun("$99").SetFontSize(28).SetFontWeight(FontWeight.Bold).SetColor(PlusUiDefaults.AccentSuccess))
    .AddRun(new TextRun(".99").SetFontSize(16).SetColor(PlusUiDefaults.AccentSuccess))
    .AddRun(new TextRun(" /month"));

// Reactive runs from the VM
new RichTextLabel().BindRuns(() => vm.Highlights);
```

**Gotchas**
- Run properties are nullable: `null` inherits from the **parent label**, not theme defaults.
- Newlines/control characters in run text are stripped automatically — pass clean text.
- `MaxLines` truncates by dropping whole lines; there is no ellipsis or visual cue.
- Runs flow inline left-to-right; there is no vertical stacking inside one label — use a `VStack` of labels for that.
- `SetRuns()` clears existing runs first; use `AddRun()` to append.
- `NoWrap` keeps everything on one line and can overflow the available width.

---

## Link

Clickable hyperlink that opens `Url` via the platform service, drawing an automatic underline. Extends `UiTextElement`; `IsFocusable => true`, `AccessibilityRole.Link`. Inherits all `UiTextElement` text properties (see Label).

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Url` | `string?` | `null` | Opened on click. Must be non-empty or click is a no-op. |
| `UnderlineThickness` | `float` | `1f` | Underline stroke width. Underline cannot be disabled. |
| `Text` | `string?` | `null` | Required — a Link with no text is invisible. |
| `TextColor` | `Color` | `White` | **No default link color** — set one explicitly. |
| `FontWeight` | `FontWeight` | `Regular` | |
| `HorizontalTextAlignment` | enum | `Left` | Also shifts the underline. |
| `TextWrapping` / `MaxLines` / `TextTruncation` | — | — | Same semantics as Label. |
| `TabIndex` | `int?` | `null` | Tab order. |
| `TabStop` | `bool` | `true` | In tab navigation by default. |
| `Margin` / `HorizontalAlignment` / `VerticalAlignment` / `IsVisible` / `Background` | inherited | — | All have `Bind*`. |

```csharp
// Minimum: text + url (style it to look like a link!)
new Link()
    .SetText("Visit Website")
    .SetUrl("https://example.com")
    .SetTextColor(Colors.CornflowerBlue);

// Styled
new Link()
    .SetText("Documentation")
    .SetUrl("https://docs.example.com")
    .SetTextColor(PlusUiDefaults.AccentPrimary)
    .SetFontWeight(FontWeight.SemiBold)
    .SetUnderlineThickness(2);

// Bound url/visibility
new Link().SetText("Profile").BindUrl(() => vm.ProfileUrl).BindIsVisible(() => vm.IsLoggedIn);
```

**Gotchas**
- **No default link-blue** — without `SetTextColor` the link renders white. Use `Colors.Blue`, `Colors.CornflowerBlue`, or `PlusUiDefaults.AccentPrimary`.
- **Empty/null `Url` makes click do nothing** (`InvokeCommand` checks `IsNullOrEmpty`).
- Underline is always drawn; only thickness is adjustable — there is no underline-less Link.
- Under wrapping, the underline is drawn per line, not as one continuous rule.
- Always focusable and a tab stop — you cannot disable focus. Use a `Label` with a click handler if you need a non-focusable clickable text.
- `Url` must be a platform-valid scheme (e.g. `https://`); opening goes through `IPlatformService.OpenUrl`.

---

## Entry

Text input with cursor navigation, selection, clipboard (Ctrl+A/C/X/V), password masking, and optional multi-line mode. Extends `UiTextElement`; `IsFocusable => true`, `AccessibilityRole.TextInput`. Defaults: width 200, height 40, `BackgroundInput` background, `CornerRadius`, padding `Margin(12, 8)`.

| Property | Type | Default | Notes |
|----------|------|---------|-------|
| `Text` | `string?` | `null` | Usually two-way bound. |
| `Placeholder` | `string?` | `null` | Shown when empty. |
| `PlaceholderColor` | `Color` | `TextPlaceholder` | |
| `IsPassword` | `bool` | `false` | Masks displayed text. |
| `PasswordChar` | `char` | `'•'` | Mask glyph. |
| `MaxLength` | `int?` | `null` | Blocks typing past limit; truncates paste. |
| `Keyboard` | `KeyboardType` | `Default` | Mobile: Default/Text/Numeric/Email/Telephone/Url. |
| `ReturnKey` | `ReturnKeyType` | `Default` | Mobile return-key label (Go/Send/Search/Next/Done). |
| `IsMultiLine` | `bool` | `false` | Enter inserts newline. Auto-sets Min/MaxLines=5 if unset. |
| `MinLines` | `int` | `1` (5 if multi-line) | Clamped to ≥1. |
| `MaxLines` | `int?` | `null` (5 if multi-line) | Scrolls beyond this. |
| `Padding` | `Margin` | `Margin(12, 8)` | Internal padding (a `Margin`, not a float). |
| `FocusBorderColor` | `Color` | `AccentPrimary` | Border when focused. |
| `FocusBorderThickness` | `float` | `2` | |
| `SelectionColor` | `Color` | `Color(100,149,237,100)` | Selection highlight (semi-transparent). |
| `TextWrapping` | `TextWrapping` | `NoWrap` | Shadows the base property. |
| `TextSize` / `TextColor` | `float` / `Color` | `12` / `White` | Inherited. |
| `Background` / `CornerRadius` / `Margin` | inherited | input theme | |

`SetOnTextChanged(Action<string?>)` fires on every type/paste/delete (no `Bind` counterpart — it is a callback, not a property).

```csharp
// Two-way bound (most common): getter expression + setter action
new Entry()
    .SetPlaceholder("Enter username...")
    .SetDesiredWidth(300)
    .BindText(() => vm.Username, v => vm.Username = v);

// Password
new Entry()
    .SetIsPassword(true)
    .SetPasswordChar('*')
    .SetMaxLength(20)
    .SetPlaceholder("Password")
    .BindText(() => vm.Password, v => vm.Password = v);

// Multi-line: set Min/MaxLines BEFORE SetIsMultiLine to keep custom values
new Entry()
    .SetMinLines(3)
    .SetMaxLines(10)
    .SetIsMultiLine(true)
    .SetDesiredWidth(400)
    .SetPlaceholder("Type notes here...")
    .BindText(() => vm.Notes, v => vm.Notes = v);

// Mobile keyboard hints
new Entry()
    .SetKeyboard(KeyboardType.Email)
    .SetReturnKey(ReturnKeyType.Next)
    .SetPlaceholder("email@example.com")
    .BindText(() => vm.Email, v => vm.Email = v);

// Live search via callback
new Entry()
    .SetPlaceholder("Search...")
    .SetPadding(new Margin(12, 8))
    .SetOnTextChanged(text => vm.SearchQuery = text);
```

**Two `BindText` overloads:**
- `BindText(Expression<Func<string?>>, Action<string>)` — string two-way.
- `BindText<T>(Expression<Func<T>>, Action<T>, Func<T,string?>? toControl = null, Func<string,T>? toSource = null)` — typed with optional converters (e.g. `int` <-> string).

**Gotchas**
- **Order matters for multi-line:** `SetIsMultiLine(true)` auto-sets MinLines=5/MaxLines=5 *if not already set* — call `SetMinLines`/`SetMaxLines` first for custom values.
- Enabling multi-line resets `DesiredHeight` (height becomes `-1`) so it can size from Min/MaxLines — don't rely on `DesiredHeight` there.
- `SetPadding` takes a `Margin` (`Margin(left,top,right,bottom)` or `Margin(horizontal,vertical)`), never a single float.
- `MaxLength` silently truncates pasted text to fit; no warning.
- Password masking uses `'•'` by default; set `PasswordChar` to override.
- Cursor blink is hardcoded at 500 ms on / 500 ms off.
- Selection is cleared on blur; single-line scrolls horizontally and multi-line shows an auto vertical scrollbar (not optional).
- `ReturnKey`/`Keyboard` only affect mobile platforms; desktop uses system defaults.
- Make `SelectionColor` opaque enough to read against custom backgrounds.

---

## Common LLM mistakes

- Using `BindText(nameof(vm.X), () => vm.X)` — **not a real overload.** Bind takes only the expression: `BindText(() => vm.X)` (read-only) or `BindText(() => vm.X, v => vm.X = v)` (Entry two-way).
- Expecting text to wrap from `SetTextWrapping` alone — you **must** also `SetDesiredWidth`.
- Forgetting `SetTextColor` on a `Link` and getting invisible/white "links."
- Setting `MaxLines` on a `Label`/`RichTextLabel` in `NoWrap` mode (ignored).
- Passing a float to `Entry.SetPadding` — it requires a `Margin`.
- Calling `SetIsMultiLine(true)` before `SetMinLines`/`SetMaxLines` and being surprised by the 5/5 defaults.
- Assuming `RichTextLabel` run `null` properties fall back to theme defaults — they inherit from the **parent label**.
- Using `SetRuns()` expecting it to append — it clears first; use `AddRun()`.
- Confusing `HorizontalTextAlignment` (text within bounds) with `HorizontalAlignment` (element in parent).
