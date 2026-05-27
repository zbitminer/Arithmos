# Arithmos — Project Sitemap & Architecture

> Gematria Calculator / Scanner — WPF Desktop Application (.NET 5, C#, MVVM)

---

## Solution Structure

```
Arithmos.sln
├── Arithmos                  (WPF UI — Views)
├── ArithmosViewModels        (ViewModels + Services + Messages)
├── ArithmosModels            (Domain Models + Business Logic)
├── ArithmosDAL               (Data Access Layer — SQLite)
└── ArithmosTests             (Unit Tests)
```

---

## 1. Arithmos — UI Layer (Views)

```
Arithmos/
├── App.xaml / App.xaml.cs          Entry point; application bootstrap
├── MainWindow.xaml / .cs           Shell window; hosts TabControl with all tabs
├── BindingAssistant.cs             Helper for proxy bindings in DataGrid columns
├── alpha.ico                       Application icon
├── arithmosdb.sqlite               Local SQLite database (dev copy)
│
├── Converters/
│   ├── AlphabetConverter           IValueConverter — Alphabet flags ↔ CheckBox
│   ├── CalculationMethodConverter  IValueConverter — CalculationMethod flags ↔ CheckBox
│   ├── InverseBooleanConverter     IValueConverter — bool → !bool
│   └── PhraseSeparatorConverter    IValueConverter — PhraseSeparator flags ↔ CheckBox
│
└── Properties/
    └── (assembly info, resources)
```

### Tab Layout (MainWindow.xaml)

```
MainWindow
└── TabControl (TabStripPlacement=Left)
    ├── [0] Calculator Tab   → DataContext: ChildViews[0] = CalculatorViewModel
    ├── [1] Scanner Tab      → DataContext: ChildViews[1] = ScannerViewModel
    ├── [2] Explorer Tab     → DataContext: ChildViews[2] = ExplorerViewModel
    ├── [3] Settings Tab     → DataContext: ChildViews[3] = SettingsViewModel
    └── [4] About Tab        → DataContext: AboutViewModel (inline)
```

---

## 2. ArithmosViewModels — ViewModel Layer

```
ArithmosViewModels/
├── MainViewModel.cs            Root VM; owns ChildViews collection (indexes 0–3)
├── CommonViewModel.cs          Abstract base VM; shared commands & state
├── CalculatorViewModel.cs      Extends CommonViewModel
├── ScannerViewModel.cs         Extends CommonViewModel
├── ExplorerViewModel.cs        Extends CommonViewModel
├── SettingsViewModel.cs        Wraps ISettingsService; auto-saves on change
├── AboutViewModel.cs           App version + GitHub link command
├── PhraseViewModel.cs          Wraps Phrase struct; adds IsSelected / IsMarked
├── OperationViewModel.cs       Wraps Operation model; adds IsSelected
│
├── Services/
│   ├── ISettingsService.cs         Interface for settings
│   ├── SettingsService.cs          Reads/writes app settings (ArithmosSettings)
│   ├── IPhraseDataService.cs       Interface for phrase CRUD
│   ├── PhraseDataService.cs        Delegates to ArithmosDAL.PhraseDataAccess
│   ├── IOperationDataService.cs    Interface for operation CRUD
│   └── OperationDataService.cs     Delegates to ArithmosDAL.OperationDataAccess
│
└── Messages/                       WeakReferenceMessenger payloads
    ├── NotificationMessage.cs      Info/success toast
    ├── ErrorMessage.cs             Error toast
    ├── ConfirmationMessage.cs      Yes/No dialog request (async response)
    ├── FileDialogMessage.cs        Open-file dialog request (async response)
    ├── IsBusyChangedMessage.cs     Broadcast busy state changes
    └── SettingsUpdatedMessage.cs   Broadcast settings save
```

### ViewModel Inheritance

```
ObservableObject (MVVM Toolkit)
├── MainViewModel
├── AboutViewModel
├── SettingsViewModel
├── PhraseViewModel
├── OperationViewModel
└── CommonViewModel
    ├── CalculatorViewModel
    ├── ScannerViewModel
    └── ExplorerViewModel
```

### CommonViewModel — Shared Commands

| Command                   | Description                                      |
|---------------------------|--------------------------------------------------|
| `ClearAllItemsCommand`    | Clears the Phrases collection                    |
| `AddNumericValueCommand`  | Adds a value to NumericValues filter list        |
| `RemoveNumericValueCommand` | Removes a value from NumericValues list        |
| `ClearNumericValuesCommand` | Clears all filter values                       |
| `MarkAllItemsCommand`     | Marks all phrases                                |
| `UnmarkAllItemsCommand`   | Unmarks all phrases                              |
| `MarkSelectedItemsCommand`| Marks currently selected rows                    |
| `UnmarkSelectedItemsCommand` | Unmarks currently selected rows              |
| `CopyMarkedItemsCommand`  | Copies marked phrase texts to clipboard          |
| `CancelCommand`           | Cancels active async operation via CTS           |
| `GetFolderPathCommand`    | Opens FolderBrowserDialog → sets ExportFolderPath|

### Per-ViewModel Commands

#### CalculatorViewModel
| Command                 | Description                                  |
|-------------------------|----------------------------------------------|
| `SaveMarkedItemsCommand`| Saves marked phrases to DB (no Operation)    |

#### ScannerViewModel
| Command                 | Description                                        |
|-------------------------|----------------------------------------------------|
| `ScanFileCommand`       | Scans a file for matching phrases (async)          |
| `ScanTextCommand`       | Scans pasted text for matching phrases (async)     |
| `GetFilePathCommand`    | Opens file dialog → sets FilePath                  |
| `SaveMarkedItemsCommand`| Saves marked phrases to DB linked to an Operation  |

#### ExplorerViewModel
| Command                          | Description                                      |
|----------------------------------|--------------------------------------------------|
| `LoadAllOperationsCommand`       | Loads all saved operations into grid             |
| `LoadSelectedOperationsCommand`  | Loads phrases for selected operations            |
| `LoadAllOrphansCommand`          | Loads phrases with no associated operation       |
| `SearchPhrasesCommand`           | Searches DB by value + method + alphabet         |
| `DeleteSelectedOperationsCommand`| Deletes selected operations (with confirmation)  |
| `DeleteMarkedItemsCommand`       | Deletes marked phrases (with confirmation)       |

---

## 3. ArithmosModels — Domain / Business Logic Layer

```
ArithmosModels/
├── Phrase.cs               Immutable struct; holds text + all 8 gematria values
├── Operation.cs            Groups phrases; has Id, Description, EntryDate
├── CalculationMethod.cs    [Flags] enum: None/Gematria/Ordinal/Reduced/Sumerian/Primes/Squared/MisparGadol/MisparShemi/All
├── Alphabet.cs             [Flags] enum: None/Mixed/English/Greek/Hebrew/All
├── PhraseSeparator.cs      [Flags] enum: NewLine/Comma/Semicolon/Tab/Colon/FullStop/Space/All
├── Exporter.cs             Static; formats phrases as CSV and writes to file (async)
│
└── Helpers/
    ├── CharacterHandler.cs     Normalizes text; detects Alphabet from characters
    ├── ValueMapper.cs          Maps characters → numeric values per CalculationMethod
    ├── PhraseHelper.cs         Checks if phrase matches target values; max-value pruning
    ├── FileHelper.cs           Generates timestamped file names
    └── Scanner.cs              Static; scans file or text → List<Phrase> (async, cancellable)
```

### Phrase Struct — Computed Properties

```
Phrase(originalText)
├── NormalizedText      CharacterHandler.NormalizeText(text)
├── Alphabet            CharacterHandler.GetAlphabet(text)
├── Gematria            ValueMapper.GetValue(text, Gematria)
├── Ordinal             ValueMapper.GetValue(text, Ordinal)
├── Reduced             ValueMapper.GetValue(text, Reduced)
├── Sumerian            ValueMapper.GetValue(text, Sumerian)
├── Primes              ValueMapper.GetValue(text, Primes)
├── Squared             ValueMapper.GetValue(text, Squared)
├── MisparGadol         ValueMapper.GetValue(text, MisparGadol)
└── MisparShemi         ValueMapper.GetValue(text, MisparShemi)
```

### Calculation Methods

| Method       | Description                                      |
|--------------|--------------------------------------------------|
| Gematria     | Standard — traditional letter values             |
| Ordinal      | A=1, B=2 … Z=26 (sequential position)           |
| Reduced      | Ordinal reduced to single digit                  |
| Sumerian     | Ordinal × 6                                      |
| Primes       | Each letter mapped to its prime number           |
| Squared      | Standard values squared                          |
| MisparGadol  | Hebrew final-form letters get elevated values    |
| MisparShemi  | Hebrew full-spelling of each letter name         |

---

## 4. ArithmosDAL — Data Access Layer

```
ArithmosDAL/
├── Database.cs             Static; holds SQLite ConnectionString
│                           Path: %LOCALAPPDATA%\ArithmosData\arithmosdb.sqlite
├── PhraseDataAccess.cs     CRUD for Phrase table (SQLite, async)
└── OperationDataAccess.cs  CRUD for Operation table (SQLite, async)
```

### Database Schema

```
Operation
├── Id          INTEGER PRIMARY KEY
├── Description TEXT
└── EntryDate   DATETIME

Phrase
├── Id               INTEGER PRIMARY KEY
├── Text             TEXT UNIQUE
├── GematriaValue    INTEGER
├── OrdinalValue     INTEGER
├── ReducedValue     INTEGER
├── SumerianValue    INTEGER
├── PrimesValue      INTEGER
├── SquaredValue     INTEGER
├── MisparGadolValue INTEGER
├── MisparShemiValue INTEGER
├── Alphabet         INTEGER  (enum cast)
└── OperationId      INTEGER  FK → Operation.Id (nullable)
```

### PhraseDataAccess Methods

| Method                                              | Description                                      |
|-----------------------------------------------------|--------------------------------------------------|
| `CreateAsync(Phrase)`                               | Insert single phrase                             |
| `CreateAsync(List<Phrase>, CancellationToken)`      | Bulk insert (transaction)                        |
| `CreateAsync(List<Phrase>, Operation, CancellationToken)` | Bulk insert + create linked Operation      |
| `RetrieveAsync(values, method, alphabet)`           | Search by gematria values + filters              |
| `RetrieveAsync(List<Operation>)`                    | Load phrases for given operations                |
| `RetrieveOrphansAsync()`                            | Load phrases with no OperationId                 |
| `DeleteAsync(List<Phrase>)`                         | Delete phrases by Id                             |

### OperationDataAccess Methods

| Method                          | Description                    |
|---------------------------------|--------------------------------|
| `CreateAsync(Operation, con)`   | Insert operation (shared conn) |
| `RetrtieveAllAsync()`           | Get all operations             |
| `DeleteAsync(List<Operation>)`  | Delete operations by Id        |

---

## 5. Data & Message Flow

### Calculator Tab Flow
```
User types text in DataGrid
  → PhraseViewModel.PhraseUserText setter
    → new Phrase(text) — auto-calculates all 8 values
      → DataGrid columns update via bindings
User clicks "Save Marked"
  → CalculatorViewModel.SaveMarkedItemsAsync()
    → PhraseDataService.CreateAsync(markedPhrases)
      → PhraseDataAccess.CreateAsync(phrases, CancellationToken)
        → SQLite INSERT OR IGNORE INTO Phrase
```

### Scanner Tab Flow
```
User selects file / pastes text
  → ScannerViewModel.ScanFileAsync() / ScanTextAsync()
    → Scanner.ScanFileAsync() / ScanTextAsync()
      → CharacterHandler.NormalizeText()
      → ValueMapper.GetValue() × 8 methods
      → PhraseHelper.PhraseContainsAnyValue() — filter match
        → Results → Phrases collection → DataGrid
          → (optional) Exporter.ExportAsync() → CSV file
User clicks "Save Marked" (with description)
  → PhraseDataService.CreateAsync(phrases, Operation, CancellationToken)
    → OperationDataAccess.CreateAsync() + PhraseDataAccess bulk insert
```

### Explorer Tab Flow
```
Tab Loaded event
  → ExplorerViewModel.LoadAllOperationsAsync()
    → OperationDataService.RetrieveAllAsync() → Operations grid

User sets filters + clicks "Search"
  → ExplorerViewModel.SearchPhrasesAsync()
    → PhraseDataService.RetrieveAsync(values, method, alphabet)
      → PhraseDataAccess.RetrieveAsync() → SQL WHERE clause
        → Results → Phrases collection → DataGrid

User selects operations + clicks "Load Selected"
  → PhraseDataService.RetrieveAsync(operations)

User clicks "Delete Selected" / "Delete Marked"
  → ConfirmationMessage sent via WeakReferenceMessenger
    → MainWindow shows dialog → user confirms
      → DAL delete → UI refresh
```

### Settings Flow
```
SettingsViewModel wraps ISettingsService
  → PropertyChanged on any setting
    → SettingsService.Save() (persists to ArithmosSettings)
      → SettingsUpdatedMessage broadcast
        → All CommonViewModels reload SettingsService
          → Column visibility / default selections update
```

### Messaging (WeakReferenceMessenger)

| Message                  | Sender                        | Receiver / Effect                        |
|--------------------------|-------------------------------|------------------------------------------|
| `NotificationMessage`    | All ViewModels                | MainWindow → shows info toast/dialog     |
| `ErrorMessage`           | All ViewModels (catch blocks) | MainWindow → shows error dialog          |
| `ConfirmationMessage`    | ExplorerViewModel             | MainWindow → shows Yes/No dialog         |
| `FileDialogMessage`      | ScannerViewModel              | MainWindow → opens OpenFileDialog        |
| `IsBusyChangedMessage`   | CommonViewModel               | ExplorerViewModel → refreshes commands   |
| `SettingsUpdatedMessage` | SettingsViewModel             | CommonViewModels → reload settings       |

---

## 6. Supported Alphabets & Languages

| Alphabet | Characters Supported                        |
|----------|---------------------------------------------|
| English  | A–Z, a–z (with diacritic normalization)     |
| Greek    | Α–Ω, α–ω (with diacritic normalization)     |
| Hebrew   | א–ת including final forms (ך ם ן ף ץ)       |
| Mixed    | Phrases containing characters from multiple alphabets |

---

## 7. Output Options (Scanner & Explorer)

| Option          | Description                                              |
|-----------------|----------------------------------------------------------|
| Show on Grid    | Populates the DataGrid in the UI                         |
| Export to file  | Writes CSV to a user-selected folder (timestamped name)  |

---

## 8. Project Dependencies

| Package                          | Used In              | Purpose                          |
|----------------------------------|----------------------|----------------------------------|
| Microsoft.Toolkit.Mvvm           | ViewModels           | ObservableObject, RelayCommand, WeakReferenceMessenger |
| System.Data.SQLite               | DAL                  | SQLite database access           |
| Microsoft.Xaml.Behaviors.Wpf     | UI (MainWindow.xaml) | EventTrigger / InvokeCommandAction |
| System.Windows.Forms             | ViewModels           | FolderBrowserDialog, OpenFileDialog |
