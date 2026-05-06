# MarkEdit for Windows — Feature Specification

> **Goal:** A native Windows port of [MarkEdit](https://github.com/MarkEdit-app/MarkEdit) built with WinUI 3 (Windows App SDK) and WebView2, preserving the project's core philosophy: *small, fast, native, correct*.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Distribution & Packaging](#2-distribution--packaging)
3. [Core Editor (CodeMirror 6 via WebView2)](#3-core-editor-codemirror-6-via-webview2)
4. [Document Model](#4-document-model)
5. [User Interface](#5-user-interface)
6. [Themes & Appearance](#6-themes--appearance)
7. [Settings & Preferences](#7-settings--preferences)
8. [Windows-Native Integrations](#8-windows-native-integrations)
9. [Automation & Scripting](#9-automation--scripting)
10. [Extension System](#10-extension-system)
11. [Accessibility & Internationalization](#11-accessibility--internationalization)
12. [Milestones](#12-milestones)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────┐
│  WinUI 3 Shell (C# / .NET 8+)              │
│  ┌───────────┐ ┌──────────┐ ┌────────────┐  │
│  │ Toolbar   │ │ Settings │ │ Status Bar │  │
│  └───────────┘ └──────────┘ └────────────┘  │
│  ┌─────────────────────────────────────────┐ │
│  │  WebView2                               │ │
│  │  ┌─────────────────────────────────────┐│ │
│  │  │ CoreEditor (TypeScript/CodeMirror 6)││ │
│  │  │ — reused from macOS MarkEdit        ││ │
│  │  └─────────────────────────────────────┘│ │
│  └─────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │ Native Bridge (WebView2 ↔ C# host)     │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
  ┌──────────────────────────────────────┐
  │ Separate Native Components (optional)│
  │ • Explorer Context Menu (COM)        │
  │ • Preview Handler (COM/C++)          │
  │ • CLI (markedit.exe)                 │
  └──────────────────────────────────────┘
```

**Key decisions:**
- **Language:** C# with .NET 8+ and Windows App SDK (WinUI 3)
- **Editor core:** Reuse the existing `CoreEditor/` TypeScript project; render inside a WebView2 control
- **Native bridge:** WebView2 `CoreWebView2.PostWebMessageAsJson` / `AddHostObjectToScript` for bidirectional communication between C# host and CodeMirror
- **Shell extensions:** Separate native COM components (C++ or C#/AOT) for Explorer context menu and preview handler — *not* hosted inside the WinUI 3 process

---

## 2. Distribution & Packaging

| Aspect | Decision |
|---|---|
| **Package format** | MSIX (primary), with optional WinGet manifest |
| **Target OS** | Windows 10 19041+ and Windows 11 |
| **Store** | Microsoft Store (optional), sideload via GitHub releases |
| **Architecture** | x64 and ARM64 |
| **Self-contained** | Yes — bundle .NET runtime to avoid dependency |
| **Auto-update** | MSIX auto-update via App Installer, or custom update check for sideloaded installs |
| **Installer size target** | < 30 MB (mirrors the "lightweight" philosophy) |

**Rationale:** MSIX packaging enables declarative file associations, protocol activation, Jump Lists, and app identity — all required for native integrations. Sideload support ensures parity with the current macOS GitHub-releases distribution.

---

## 3. Core Editor (CodeMirror 6 via WebView2)

Reuse the existing `CoreEditor/` TypeScript codebase with minimal Windows-specific adaptations.

### 3.1 Editor Features (parity with macOS)

| Feature | Source | Notes |
|---|---|---|
| Markdown syntax highlighting | `CoreEditor/src/core.ts` | GFM + YAML frontmatter via `@lezer/markdown` |
| Bracket matching & auto-close | CodeMirror built-in | |
| Code folding | CodeMirror folding keymap | |
| Multi-caret editing | CodeMirror multi-selection | |
| Rectangular selection | `crosshairCursor` extension | |
| Undo / redo | CodeMirror history | |
| Active line highlight | `highlightActiveLine` | |
| Line numbers | `lineNumbers` gutter | |
| Invisible characters | Custom extension | |
| Typewriter / focus mode | Custom extension | |
| Word wrap | `EditorView.lineWrapping` | |

### 3.2 Find & Replace

| Capability | Notes |
|---|---|
| Case-sensitive toggle | |
| Diacritic-insensitive toggle | Custom normalizer |
| Whole-word matching | |
| Literal matching | |
| Regular expressions | |
| Find next / previous | |
| Replace next / Replace all | |
| Select all occurrences | |
| Select next occurrence | |

### 3.3 Formatting Commands

| Command | Shortcut (Windows) |
|---|---|
| Bold | `Ctrl+B` |
| Italic | `Ctrl+I` |
| Strikethrough | `Ctrl+Shift+X` |
| Inline code | `Ctrl+E` |
| Inline math | `Ctrl+Shift+E` |
| Code block | `Ctrl+Shift+C` |
| Math block | `Ctrl+Shift+M` |
| Heading 1–6 | `Ctrl+1` – `Ctrl+6` |
| Blockquote | `Ctrl+Shift+.` |
| Bullet list | `Ctrl+Shift+8` |
| Numbered list | `Ctrl+Shift+7` |
| Todo list | `Ctrl+Shift+9` |
| Horizontal rule | — |
| Toggle comment | `Ctrl+/` |
| Move line up/down | `Alt+Up/Down` |
| Copy line | `Alt+Shift+Up/Down` |
| Indent / outdent | `Tab` / `Shift+Tab` |

### 3.4 Table of Contents & Navigation

- Extract headings from document → display in a sidebar/flyout panel
- Navigate to heading via keyboard (`Ctrl+Shift+O`)
- Section-by-section navigation

### 3.5 Autocomplete

| Source | Trigger |
|---|---|
| Words in document | Typing (configurable) |
| Link references | `[` |
| Footnotes | `[^` |
| File paths | Typing in link target |
| Inline prediction acceptance | `Tab` |

### 3.6 WebView2 Bridge

The native bridge replaces the macOS `WKWebView` message handler layer:

```
C# Host ←→ WebView2 CoreWebView2
  │                    │
  ├── postMessage ────→│  (host → editor: config, commands, text ops)
  │←── postMessage ────┤  (editor → host: events, state, completions)
  │                    │
  └── addHostObject ──→│  (expose native APIs to JS: file I/O, clipboard)
```

Bridge modules to implement (mirroring `CoreEditor/src/bridge/`):
- **Core:** theme changes, config updates, text get/set, focus, scroll
- **Search:** find/replace operations, highlight management
- **Format:** formatting command dispatch
- **Completion:** autocomplete data source, inline prediction
- **Selection:** multi-caret state, selection info for status bar
- **TOC:** heading list extraction for navigation panel
- **Events:** content change, save request, dirty state, focus/blur

---

## 4. Document Model

> *Replaces macOS `NSDocument`. This is the most significant new implementation area.*

### 4.1 File Lifecycle

| Feature | Behavior |
|---|---|
| **Open** | Standard Windows file picker; also via file association, CLI, protocol, drag-and-drop, recent files |
| **Save / Save As** | Atomic write (temp → flush → rename); preserve original encoding and line endings |
| **Autosave** | Periodic autosave to a recovery file (e.g., `~/.markedit/recovery/`); configurable interval; restore on crash |
| **Dirty state** | Track via CodeMirror change events; show in title bar (`● filename.md`); confirm on close |
| **Revert** | Reload from disk; confirm if document is dirty |
| **External modification** | Detect via `FileSystemWatcher`; prompt user to reload or keep |
| **Conflict resolution** | If file changed on disk while also dirty in editor, offer: reload, overwrite, or save as new |

### 4.2 Encoding & Line Endings

| Feature | Details |
|---|---|
| **Encoding detection** | Auto-detect on open (UTF-8, UTF-8 BOM, UTF-16 LE/BE, Windows-1252, etc.) |
| **Encoding selection** | Configurable default; override via Save As dialog |
| **Line endings** | Detect on open (CRLF, LF, CR); preserve original; configurable default for new files |
| **Final newline** | Option to ensure file ends with a newline on save |
| **Trim trailing whitespace** | Option to strip trailing whitespace on save |

### 4.3 Supported File Types

| Type | Extensions | Notes |
|---|---|---|
| Markdown | `.md`, `.markdown`, `.mdown`, `.mkd`, `.mdx` | Primary |
| Text | `.txt`, `.text` | Plain text mode |
| TextBundle | `.textbundle` | Directory-based Markdown package |
| Other text formats | Configurable | Open any text file with limited highlighting |

### 4.4 Recent Documents

- Integrate with Windows **Jump List** (recent files category)
- Maintain an internal MRU list for the app's "Open Recent" menu
- Pinned files support in Jump List

---

## 5. User Interface

### 5.1 Window Chrome

| Element | Implementation |
|---|---|
| **Title bar** | Custom title bar with Mica backdrop; show filename, dirty indicator, and path |
| **Toolbar** | WinUI `CommandBar` — formatting buttons, search toggle, TOC toggle, settings |
| **Tab bar** | Optional tabbed interface via `TabView`; configurable (tabs vs. separate windows) |
| **Status bar** | Bottom bar: line number, column, selection count, character count, encoding, line ending indicator |
| **Content area** | WebView2 filling the remaining space |

### 5.2 Find & Replace Panel

- Inline panel at top of editor (similar to VS Code / macOS MarkEdit)
- Toggle with `Ctrl+F` (find) / `Ctrl+H` (replace)
- Options: case, diacritics, whole word, literal, regex — as toggle buttons
- Result count indicator
- Close with `Escape`

### 5.3 Table of Contents Panel

- Side panel or flyout, toggled via toolbar button or `Ctrl+Shift+O`
- Hierarchical heading list with indentation by level
- Click to navigate; highlight current heading
- Filter/search within TOC

### 5.4 Settings UI

- WinUI `NavigationView` with pages:
  - **General** — appearance, new window behavior, default encoding, line endings, file associations
  - **Editor** — theme, font, line numbers, active line, invisibles, typewriter mode, word wrap, line height, tab/indent
  - **Writing** — final newline, trim whitespace, inline predictions, suggest-while-typing
  - **Search** — default search options
  - **Window** — toolbar mode, tab mode, Mica/Acrylic preference
  - **Advanced** — recovery/autosave, extensions directory, CLI integration
- Settings stored in `settings.json` at `%APPDATA%/MarkEdit/` (portable)
- Runtime reload without restart where possible

### 5.5 Menus

| Menu | Key Items |
|---|---|
| **File** | New, Open, Open Recent ▸, Save, Save As, Revert, Print, Export HTML/PDF |
| **Edit** | Undo, Redo, Cut, Copy, Paste, Find ▸, Select All |
| **Format** | All formatting commands (bold, italic, headings, lists, etc.) |
| **View** | Toggle line numbers, invisibles, word wrap, typewriter mode, focus mode, TOC panel, zoom |
| **Window** | New Window, Tabs, Minimize, Close |
| **Help** | About, Documentation, Check for Updates, Keyboard Shortcuts |

### 5.6 Dialogs

- **Save options:** Encoding selector, line ending selector, file extension override (in Save As)
- **Go to line:** `Ctrl+G` — jump to line/column
- **About:** Version, build info, links, license

### 5.7 Drag & Drop

| Action | Behavior |
|---|---|
| Drop `.md` file on window | Open file in current or new tab |
| Drop image file onto editor | Copy to document folder, insert `![](relative-path)` |
| Drop text onto editor | Insert at drop position |
| Paste image from clipboard | Save to document folder, insert Markdown image link |

---

## 6. Themes & Appearance

### 6.1 Built-in Themes

Port all 10 macOS themes (light + dark variants each):

| Theme | Light | Dark |
|---|---|---|
| GitHub | ✓ | ✓ |
| Xcode | ✓ | ✓ |
| Dracula | — | ✓ |
| Cobalt | — | ✓ |
| Winter | ✓ | — |
| Minimal | ✓ | ✓ |
| SynthWave | — | ✓ |
| Night Owl | — | ✓ |
| Rosé Pine | ✓ | ✓ |
| Solarized | ✓ | ✓ |

### 6.2 System Integration

| Feature | Details |
|---|---|
| **App theme** | Follow system (default), force Light, force Dark |
| **Mica / Acrylic** | Mica on Windows 11, Acrylic fallback on Windows 10; user can disable |
| **Accent color** | Respect system accent color for selection highlights, toolbar |
| **High Contrast** | Full support — detect and apply high-contrast-safe styles |
| **Reduce transparency** | Respect system setting; disable backdrop materials |

### 6.3 Custom Themes

- Users can place custom CSS in `%APPDATA%/MarkEdit/themes/`
- Theme = CSS file targeting CodeMirror classes
- Reuse the macOS [MarkEdit-theming](https://github.com/MarkEdit-app/MarkEdit-theming) extension format

---

## 7. Settings & Preferences

### 7.1 General

| Setting | Type | Default |
|---|---|---|
| Appearance | System / Light / Dark | System |
| New window behavior | Empty / Last file / Open dialog | Empty |
| Default file extension | `.md` / `.markdown` / `.txt` | `.md` |
| Default text encoding | UTF-8 / UTF-8 BOM / others | UTF-8 |
| Default line endings | CRLF / LF / System | CRLF |
| Show hidden files in dialogs | Boolean | false |
| Restore windows on launch | Boolean | true |

### 7.2 Editor

| Setting | Type | Default |
|---|---|---|
| Theme (light) | Theme enum | GitHub Light |
| Theme (dark) | Theme enum | GitHub Dark |
| Font family | String | System monospace |
| Font size | Number | 14 |
| Show line numbers | Boolean | true |
| Highlight active line | Boolean | true |
| Show selection status | Boolean | true |
| Show invisibles | Boolean | false |
| Typewriter mode | Boolean | false |
| Focus mode | Boolean | false |
| Word wrap | Boolean | true |
| Line height | Number | 1.5 |
| Use tabs for indentation | Boolean | false |
| Indent unit (spaces) | Number | 4 |

### 7.3 Writing Assistance

| Setting | Type | Default |
|---|---|---|
| Ensure final newline | Boolean | true |
| Trim trailing whitespace | Boolean | false |
| Suggest while typing | Boolean | true |
| Inline predictions | Boolean | true |
| Spellcheck | Boolean | true |
| Spellcheck language | System / specific | System |
| Auto character pairs | Boolean | true |

### 7.4 Search Defaults

| Setting | Type | Default |
|---|---|---|
| Case sensitive | Boolean | false |
| Diacritic insensitive | Boolean | false |
| Whole word | Boolean | false |
| Literal | Boolean | false |
| Regular expression | Boolean | false |

### 7.5 Window

| Setting | Type | Default |
|---|---|---|
| Toolbar mode | Icon / Text / Both / Hidden | Icon |
| Tabbing mode | Tabs / Windows | Tabs |
| Background material | Mica / Acrylic / None | Mica |

---

## 8. Windows-Native Integrations

### 8.1 File Type Associations

- Register as handler for `.md`, `.markdown`, `.mdown`, `.mkd`, `.mdx`, `.textbundle`
- Declared in MSIX `Package.appxmanifest` via `<uap:FileTypeAssociation>`
- Include document icons (light + dark variants)
- Open files in existing app instance (single-instance activation)

### 8.2 Protocol Activation

| URI | Action |
|---|---|
| `markedit://new` | Create new document |
| `markedit://new?filename=X&content=Y` | Create new document with content |
| `markedit://open?path=C:\...\file.md` | Open file at path |
| `markedit://open?path=...&line=42` | Open file and go to line |

**Security:** Validate all parameters. Prompt user before opening paths received via protocol. Never execute arbitrary code from URI parameters.

### 8.3 Jump List

| Category | Items |
|---|---|
| **Recent** | Last 10 opened files (auto-managed via `JumpList` API) |
| **Tasks** | "New Document", "New Window" |
| **Pinned** | User-pinnable documents |

### 8.4 Single-Instance & Activation Redirection

- Use `AppInstance.FindOrRegisterForKey` and `AppInstance.RedirectActivationToAsync` from Windows App SDK
- When a second instance launches (file association, protocol, CLI), redirect to existing instance
- Existing instance opens file in new tab or window (per user preference)

### 8.5 Global Hotkey

- User-configurable keyboard shortcut to activate MarkEdit or create new document
- Default: none (must be explicitly set by user)
- Implemented via `RegisterHotKey` Win32 interop
- Detect registration failure and notify user of conflicts
- Respect when app is minimized to system tray (optional tray icon)

### 8.6 Snap Layouts & Window Management

- Support Windows 11 Snap Layouts (automatic with proper WinUI 3 usage)
- Remember window position/size per monitor configuration
- Multi-window support with independent documents

### 8.7 Notifications

- Toast notification on autosave recovery ("Recovered unsaved changes from crash")
- Optional: notification when background export completes

### 8.8 Printing & Export

| Action | Implementation |
|---|---|
| **Print** | Render Markdown to HTML → WebView2 print API |
| **Export HTML** | Render to styled HTML file |
| **Export PDF** | WebView2 `PrintToPdfAsync` |

### 8.9 Spellcheck

- Use WebView2's built-in spellcheck (backed by Windows spellchecking service)
- Language follows system or user override
- Custom dictionary support at `%APPDATA%/MarkEdit/dictionary.txt`

### 8.10 Explorer Context Menu *(Phase 2)*

- "New Markdown file" entry in Explorer right-click → New menu
- "Open with MarkEdit" for supported file types
- Implemented as a packaged COM-based `IExplorerCommand` (separate native component)
- Registered via MSIX sparse package or app package extension

### 8.11 Preview Handler *(Phase 2)*

- Markdown preview in Explorer preview pane and Outlook
- Implemented as a separate COM-based `IPreviewHandler` (C++ or C#/AOT)
- Renders Markdown to HTML using a lightweight renderer
- Not hosted inside the WinUI 3 process

### 8.12 Taskbar Integration *(Phase 2)*

- Progress indicator for long-running operations (large file export)
- Taskbar badge for unsaved document count (optional)

---

## 9. Automation & Scripting

### 9.1 Command-Line Interface

Primary automation surface. The `markedit.exe` CLI should support:

```
markedit                          # Launch or focus app
markedit <file>                   # Open file
markedit <file>:<line>            # Open file at line
markedit <file> --goto <line>,<col>  # Open file at line:col
markedit --new                    # New document
markedit --new --content "text"   # New document with content
markedit --wait <file>            # Open and block until file is closed
markedit --diff <file1> <file2>   # Diff two files (future)
markedit --list-extensions        # List installed extensions
markedit --disable-extensions     # Launch in safe mode
```

### 9.2 PowerShell Module *(Phase 2)*

Wrapper around CLI/protocol for PowerShell scripting:

```powershell
Import-Module MarkEdit
Open-MarkEditFile "README.md" -Line 42
New-MarkEditDocument -Content "# Hello"
Get-MarkEditDocument  # returns current document info
```

### 9.3 Protocol Activation

See [Section 8.2](#82-protocol-activation).

---

## 10. Extension System

### 10.1 CodeMirror Extensions (Reuse from macOS)

- Users place JavaScript and CSS files in `%APPDATA%/MarkEdit/extensions/`
- Extensions are loaded into the WebView2 CodeMirror instance at startup
- Compatible with the existing [MarkEdit-api](https://github.com/MarkEdit-app/MarkEdit-api) extension format
- API surface: `addExtension()`, `addMarkdownConfig()`, `addCodeLanguage()`

### 10.2 Security Constraints

| Rule | Details |
|---|---|
| Extensions are local-only | No remote extension loading or installation via URL |
| No native API access | Extensions cannot call host objects or native APIs |
| Sandboxed WebView2 context | Extensions run inside the WebView2 content; no access to file system or network beyond what CodeMirror provides |
| Safe mode | `--disable-extensions` CLI flag to launch without extensions |
| Error isolation | Extension errors are caught and logged; they do not crash the editor |

### 10.3 Custom Styles

- CSS files in `%APPDATA%/MarkEdit/styles/` are injected into the editor
- Compatible with macOS custom style format

---

## 11. Accessibility & Internationalization

### 11.1 Accessibility

| Feature | Details |
|---|---|
| **Screen reader** | UI Automation for WinUI controls; ARIA labels in WebView2 content |
| **Keyboard navigation** | Full keyboard access to all UI; no mouse-only interactions |
| **High Contrast** | Detect and apply high-contrast theme; test with all 4 Windows HC themes |
| **Text scaling** | Respect system text-scaling setting for UI; editor font scales independently |
| **Reduced motion** | Respect `prefers-reduced-motion`; disable animations |
| **Per-monitor DPI** | Full per-monitor DPI awareness (automatic with WinUI 3) |

### 11.2 Internationalization

| Feature | Details |
|---|---|
| **UI language** | Follow system language; English-only for v1 with localization framework in place |
| **IME support** | Full IME support via WebView2 (CJK, etc.) |
| **RTL text** | Support RTL text within Markdown content |
| **Unicode** | Full Unicode support in file content and filenames |

---

## 12. Milestones

### Phase 1 — Core Editor & App Shell

> Goal: Feature-complete Markdown editing experience

- [ ] Project scaffolding (WinUI 3 + WebView2 + CoreEditor integration)
- [ ] WebView2 ↔ C# native bridge (all bridge modules)
- [ ] Document model (open, save, save-as, dirty state, encoding, line endings)
- [ ] Autosave & crash recovery
- [ ] External file change detection
- [ ] Find & Replace panel
- [ ] Table of Contents panel
- [ ] All formatting commands with Windows keybindings
- [ ] Status bar (line, column, selection, encoding, line endings)
- [ ] Settings UI (all preference categories)
- [ ] All 10 built-in themes (light + dark)
- [ ] Mica/Acrylic window chrome
- [ ] System light/dark/high-contrast theme support
- [ ] File type associations (MSIX manifest)
- [ ] Protocol activation (`markedit://`)
- [ ] Jump List (recent files + tasks)
- [ ] Single-instance activation redirection
- [ ] CLI (`markedit.exe` — open, new, goto)
- [ ] Drag-and-drop (files + images + text)
- [ ] Spellcheck (WebView2 built-in)
- [ ] Print / Export HTML / Export PDF
- [ ] TextBundle support
- [ ] Custom CSS/JS extensions (local-only)
- [ ] Accessibility baseline (keyboard, screen reader, high contrast, DPI)

### Phase 2 — Advanced Windows Integration

> Goal: Deep OS integration and automation

- [ ] Explorer context menu ("New Markdown file", "Open with MarkEdit")
- [ ] Preview handler (Markdown preview in Explorer/Outlook)
- [ ] PowerShell module
- [ ] Taskbar progress/badge
- [ ] Global hotkey
- [ ] System tray icon (optional)
- [ ] `--wait` CLI flag for editor-as-external-tool workflows
- [ ] Microsoft Store submission
- [ ] Localization (community-contributed translations)

### Phase 3 — Stretch Goals

- [ ] Diff view (`markedit --diff`)
- [ ] Windows Copilot / App Actions integration
- [ ] Pen/ink annotation overlay
- [ ] IFilter for TextBundle (Windows Search indexing)
- [ ] Thumbnail handler for Markdown files in Explorer
