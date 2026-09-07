# AGENTS.md — Biblioteca Knowledge Base

## Project

GeneXus 18.0 U15 Knowledge Base — Library Management System ("Biblioteca").  
Generator: .NET (Web). Database: SQL Server. KB language: Spanish.  
Design System: GeneXusUnanimo v2.0.194 (Chameleon web components).

## Tooling

- **gxnext CLI** (v1.0.0) is the primary tool for importing/exporting objects.
- **MCP server** (`GeneXus.PIA.McpServer.exe`) exposes `open_knowledge_base`, `import_text_to_kb`, `export_kb_to_text`, `validate_kb_text_files`, `build_one`, `build_all`, `reorganize`, `run`.
- The KB must be opened before any MCP operation: `open_knowledge_base` with `directory: "C:\KBs\IA\Prueba\Biblioteca"`.

### Key gxnext commands

```
# Import one or more objects
gxnext import-text-to-kb --names "ObjectName" [--names "Other"] --root-directory "C:\KBs\IA\Prueba"

# Export objects back to text
gxnext export-kb-to-text --names "ObjectName" --root-directory "C:\KBs\IA\Prueba"

# Validate without importing
gxnext validate-kb-text-files --names "ObjectName" --root-directory "C:\KBs\IA\Prueba"
```

Always export after import to verify GeneXus accepted the file as written (it may reformat or reject silently).

## Source Layout

```
src/
  #attributes/      Attribute definitions ([Entity][Attribute].gx)
  #tables/          Table definitions (Book, Author, Loan, Genre, Publisher, Shelf)
  #domains/         Domain definitions (ISBN, ReadingStatus)
  #designsystems/   Design system references (Biblioteca.gx, Base.gx)
  #preferences/     KB and environment config (Biblioteca.kb.gx, NET.env.gx)
  #patternsettings/ WorkWith pattern config
  Author/           WorkWithWebAuthor pattern + components
  Book/             WorkWithWebBook pattern + components
  Genre/            WorkWithWebGenre pattern + components
  Loan/             WorkWithWebLoan pattern + components
  Publisher/        WorkWithWebPublisher pattern + components
  Shelf/            WorkWithWebShelf pattern + components
  General/
    UI/             Master pages, DataProviders, SDTs, Procedures
    Security/       IsAuthorized procedure
ref/                External module references (GeneXus core, GeneXusUnanimo)
Biblioteca/         KB runtime artifacts — do not edit manually
```

New custom objects (Procedures, WebPanels, SDTs) go in `src/General/UI/` unless they belong to a specific business module.

## GeneXus Object Format (.gx files)

### Naming conventions

- **Attributes**: `[Entity][Attribute]` — e.g., `BookId`, `LoanExpectedReturnDate`
- **Objects**: PascalCase — e.g., `WorkWithWebBook`, `GetOverdueLoans`
- **Variables**: `&PascalCase` — e.g., `&BookId`, `&OverdueLoanItem`
- **Localizable strings**: prefix with `!` — e.g., `!"Préstamos Vencidos"`

### Indentation

Tabs (one per level). Match the style of existing `.gx` files exactly.

### SDT structure

- The root element must have `Collection = 'False'` explicitly set to prevent GeneXus from auto-converting it to a collection.
- Nested collection item type reference uses `SDTName.CollectionItemName` (the `CollectionItemName` attribute value), **not** the collection element name.

```gx
SDT MySDT
{
    MySDT [ Collection = 'False' ]{
        Items [ Collection = 'True', CollectionItemName = 'Item' ]{
            Field [ DataType = 'VarChar(100)' ]
        }
    }
}
// Variable type for an item: 'MySDT.Item'
```

### Procedure code — known syntax constraints

| ❌ Does NOT work in procedure code | ✅ Use instead |
|---|---|
| `Today()` | `Now()` (returns DateTime; assign to Date variable) |
| `CToD('')` | Declare a `Date` variable without initializing it (null by default) |
| `For Each TableName` | `For Each` (GeneXus infers the table from attributes used) |
| `Order by Attribute` | `Order Attribute` (no `by`) |
| `Order` after `Where` | Put `Order` **before** `Where` clauses |

`Today()` and `CToD('')` **are** valid inside `#Conditions` and `#Events` sections.

### WebPanel properties

- `MasterPage = "MasterUnanimoSidebar"` — applies the sidebar master page.
- `Style` is set automatically by GeneXus on import; do not set it manually.
- Conditions go in the `#Conditions` section, not in `Event Grid.Load`.

### Layout files (.web.xml)

Each WebPanel with a custom layout needs a companion `ObjectName.web.xml` file in the same directory. Use the `tabularGrid` pattern from existing files (e.g., `src/Book/WorkWithWebBook/WWBook.web.xml`) as a reference.

## UI Architecture

- **Master page**: `MasterUnanimoSidebar` (module `General.UI`) — all web pages use this.
- **Sidebar navigation**: `SidebarItemsDP` DataProvider — add a `SidebarItem` block to register a new page in the sidebar.
- **Authorization**: Call `IsAuthorized(&PgmName)` in `Event Start`; redirect with `NotAuthorized(&PgmName)` if false.
- **WorkWith pattern**: All CRUD UIs are generated via the WorkWith pattern. Prefer extending the pattern over creating standalone WebPanels for entity management.

## Data Model (core entities)

| Entity | Key attributes |
|---|---|
| Book | BookId, BookTitle, BookISBN, BookStatus (ReadingStatus domain), BookRating (0–5) |
| Author | AuthorId, AuthorName, AuthorBirthDate |
| Loan | LoanId, BookId (FK), LoanPersonName, LoanDate, LoanExpectedReturnDate, LoanReturnedDate |
| Genre | GenreId, GenreName |
| Publisher | PublisherId, PublisherName |
| Shelf | ShelfId, ShelfName, ShelfDescription |

`LoanReturnedDate` empty + `LoanExpectedReturnDate < today` = overdue loan.

## Module references

- `GeneXus` core module: `ref/GeneXus/` — provides base types (`ObjectName`, `Url`, `Boolean`, etc.)
- `GeneXusUnanimo` v2.0.194: `ref/GeneXusUnanimo/` — provides design system, Chameleon controls, sidebar SDTs, stencils.
- Reference objects with their module qualifier when needed: `'ObjectName, GeneXus'`, `'OverdueLoanItem, General.UI'`.
