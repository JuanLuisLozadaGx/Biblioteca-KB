# AGENTS.md — Biblioteca Knowledge Base

## Project

GeneXus 18.0 U15 Knowledge Base — Library Management System ("Biblioteca").  
Generator: .NET (Web). Database: **SQL Server 2019** (`localhost`, instancia `MSSQLSERVER`). KB language: Spanish.  
Design System: GeneXusUnanimo v2.0.194 (Chameleon web components).

## Version Control

- **Git repo**: `C:\KBs\IA\Prueba` — inicializado, rama `main`.
- **Remote**: `https://github.com/JuanLuisLozadaGx/Biblioteca-KB`
- **Tag**: `Version1` — estado inicial de la KB con módulo OverdueLoans.
- **Push**: `git push -u origin main && git push origin --tags`

## Database — SQL Server 2019

La KB fue migrada de LocalDB a SQL Server 2019. Configuración actual:

| Parámetro | Valor |
|---|---|
| Instancia | `localhost` (MSSQLSERVER) |
| Base de datos | `GX_KB_Biblioteca` |
| Autenticación | SQL Server — usuario `sa` |
| Archivos | `C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\DATA\` |
| `DetachOnClose` | `False` (no se desconecta al cerrar el IDE) |
| `CreateDbInKbFolder` | `False` (los archivos NO están en `Biblioteca/`) |

El archivo de conexión es `Biblioteca/knowledgebase.connection` — no editar manualmente salvo para cambiar el servidor.

### Para nuevas KBs — usar SQL Server 2019 desde el inicio

En el asistente de GeneXus: `File → New → Knowledge Base` → sección **Database connection**:
- Server type: Microsoft SQL Server
- Server name: `localhost`
- Authentication: SQL Server / usuario `sa`
- **No** usar LocalDB (evita bloqueos de archivos `.mdf`).

## Tooling

- **gxnext CLI** (v1.0.0) es la herramienta principal para importar/exportar objetos.
- **MCP server** (`GeneXus.PIA.McpServer.exe`) expone `open_knowledge_base`, `import_text_to_kb`, `export_kb_to_text`, `validate_kb_text_files`, `build_one`, `build_all`, `reorganize`, `run`.
- La KB debe estar abierta antes de cualquier operación MCP: `open_knowledge_base` con `directory: "C:\\KBs\\IA\\Prueba\\Biblioteca"`.

### Key gxnext commands

```
# Importar uno o más objetos
gxnext import-text-to-kb --names "ObjectName" [--names "Other"] --root-directory "C:\KBs\IA\Prueba"

# Exportar objetos a texto
gxnext export-kb-to-text --names "ObjectName" --root-directory "C:\KBs\IA\Prueba"

# Validar sin importar
gxnext validate-kb-text-files --names "ObjectName" --root-directory "C:\KBs\IA\Prueba"
```

Siempre exportar después de importar para verificar que GeneXus aceptó el archivo tal como fue escrito (puede reformatear o rechazar silenciosamente).

## Coexistencia IDE + Coda

- **Sí se puede** tener el IDE de GeneXus abierto mientras Coda edita archivos `.gx` en disco.
- **Antes de importar objetos** (`import-text-to-kb`), cerrar o guardar el objeto en el IDE para evitar conflictos.
- El archivo `.mdf` queda bloqueado por SQL Server mientras la KB está abierta — esto es normal y esperado.
- El shadow repo de Coda respeta el `.gitignore`; los archivos `.mdf/.ldf/.ndf` están excluidos.

## .gitignore — exclusiones clave

```
Biblioteca/       # runtime artifacts de la KB (incluye .mdf/.ldf)
*.mdf             # SQL Server data files (bloqueados en runtime)
*.ldf             # SQL Server log files
*.ndf             # SQL Server secondary data files
web/              # output de build/deploy
DeploymentUnit/   # output de deploy
.coda/            # workspace interno de Coda
```

## Source Layout

```
src/
  #attributes/        Definiciones de atributos ([Entidad][Atributo].gx)
  #tables/            Definiciones de tablas (Book, Author, Loan, Genre, Publisher, Shelf)
  #domains/           Dominios (ISBN, ReadingStatus)
  #designsystems/     Referencias de design system (Biblioteca.gx, Base.gx)
  #preferences/       Config de KB y entorno (Biblioteca.kb.gx, NET.env.gx)
  #patternsettings/   Config del patrón WorkWith
  Author/             Patrón WorkWithWebAuthor + componentes
  Book/               Patrón WorkWithWebBook + componentes
  Genre/              Patrón WorkWithWebGenre + componentes
  Loan/               Patrón WorkWithWebLoan + componentes
  Publisher/          Patrón WorkWithWebPublisher + componentes
  Shelf/              Patrón WorkWithWebShelf + componentes
  @OverduelLoans/     Módulo de préstamos vencidos (⚠ doble-l en el nombre, coincide con la KB)
    GetOverdueLoans.gx      Procedure — consulta préstamos vencidos, retorna SDT OverdueLoanItem
    OverdueLoanItem.gx      SDT — colección de ítems de préstamos vencidos
    OverdueLoansPanel.gx    WebPanel — grilla de préstamos vencidos
  General/
    UI/               Master pages, DataProviders, SDTs, Procedures, WebPanels custom
      DataLoad.gx           WebPanel — eventos 'LoadData' y 'ClearData' para datos de prueba
      SidebarItemsDP.gx     DataProvider — ítems del sidebar (registrar nuevas páginas aquí)
      MasterUnanimoSidebar  Master page principal
    Security/         IsAuthorized, NotAuthorized
ref/                  Referencias de módulos externos (GeneXus core, GeneXusUnanimo)
Biblioteca/           Artefactos runtime de la KB — NO editar manualmente
```

Objetos nuevos (Procedures, WebPanels, SDTs) van en `src/General/UI/` salvo que pertenezcan a un módulo de negocio específico. Los módulos creados desde el IDE de GeneXus reciben prefijo `@` en el nombre de carpeta.

## GeneXus Object Format (.gx files)

### Naming conventions

- **Atributos**: `[Entidad][Atributo]` — ej. `BookId`, `LoanExpectedReturnDate`
- **Objetos**: PascalCase — ej. `WorkWithWebBook`, `GetOverdueLoans`
- **Variables**: `&PascalCase` — ej. `&BookId`, `&OverdueLoanItem`
- **Strings localizables**: prefijo `!` — ej. `!"Préstamos Vencidos"`

### Indentación

Tabs (uno por nivel). Respetar el estilo de los `.gx` existentes exactamente.

### SDT structure

- El elemento raíz debe tener `Collection = 'False'` explícito para evitar que GeneXus lo convierta en colección.
- La referencia al tipo de ítem de colección usa `SDTName.CollectionItemName` (el valor del atributo `CollectionItemName`), **no** el nombre del elemento colección.

```gx
SDT MySDT
{
    MySDT [ Collection = 'False' ]{
        Items [ Collection = 'True', CollectionItemName = 'Item' ]{
            Field [ DataType = 'VarChar(100)' ]
        }
    }
}
// Tipo de variable para un ítem: 'MySDT.Item'
```

### Procedure code — restricciones de sintaxis conocidas

| ❌ NO funciona en procedure code | ✅ Usar en su lugar |
|---|---|
| `Today()` | `Now()` (retorna DateTime; asignar a variable Date) |
| `CToD('')` | Declarar variable `Date` sin inicializar (null por defecto) |
| `For Each TableName` | `For Each` (GeneXus infiere la tabla por los atributos usados) |
| `Order by Attribute` | `Order Attribute` (sin `by`) |
| `Order` después de `Where` | Poner `Order` **antes** de las cláusulas `Where` |
| `BookStatus = "Read"` (string a dominio enum) | `BookStatus = ReadingStatus.Read` (sintaxis `DomainName.EnumValue`) |
| `For Each Author ... EndFor` para contar | `&Count = Count(AuthorId)` — evita establecer contexto de tabla |

`Today()` y `CToD('')` **sí son válidos** dentro de secciones `#Conditions` y `#Events`.

### Contexto de tabla en bloques `New` — gotcha crítico

El especificador de GeneXus mantiene el contexto de tabla activo después de un `For Each ... EndFor` o de bloques `New`. Si en un mismo procedimiento hay bloques `New` para **distintas tablas** en secuencia, el especificador puede "heredar" el contexto de la tabla anterior y no encontrar los atributos de la tabla siguiente, creando variables Numeric automáticas.

**Síntoma**: `error spc0010: Type mismatch in assignment: PublisherName = "..." (Numeric=Character)` — GeneXus trata el atributo como variable Numeric porque no lo encuentra en el contexto actual.

**Regla**: Nunca usar `For Each TableA ... EndFor` antes de bloques `New` para otras tablas en el mismo procedimiento. Usar `Count(AttributeId)` para contar registros sin establecer contexto de tabla.

### WebPanel events — restricciones adicionales

- **`New` NO es válido en eventos de WebPanel.** Usar un Procedure para insertar registros y llamarlo desde el evento.
- **`For Each / Delete` NO es válido en eventos de WebPanel.** Ídem — delegar a un Procedure.
- **Eventos de botón** usan la sintaxis `Event 'EventName'` (literal string). El botón en el layout referencia el evento via `onClickEvent="'EventName'"` dentro de un `<actiongroup>`.
- **User events** (`Event 'Name'`) aparecen como botones de acción en el layout por defecto automáticamente.

### Layout files (.web.xml) — elementos válidos

| Elemento | Propósito |
|---|---|
| `<responsive>` | Contenedor responsive |
| `<flex>` | Contenedor flex |
| `<row>` / `<cell>` | Filas y celdas de layout |
| `<label>` | Texto estático (atributo `caption`) |
| `<input data="&amp;VarName">` | Display de variable/atributo |
| `<actiongroup name="X" />` | Placeholder para un grupo de acciones |
| `<actiongroups>` / `<actiongroup>` / `<action>` | Definición de botones (fuera de `<view>`) |
| `<tabularGrid>` | Grilla de datos |

❌ `<textblock>` NO es un elemento estándar — usar `<label>` en su lugar.  
❌ `<button>` NO es un elemento estándar — usar `<action onClickEvent="'EventName'">` dentro de `<actiongroups>`.  
❌ `<attribute>` NO es un elemento estándar — usar `<input data="&amp;VarName">` en su lugar.

### WebPanel properties

- `MasterPage = "MasterUnanimoSidebar"` — aplica la master page con sidebar.
- `Style` lo setea GeneXus automáticamente al importar; no setearlo manualmente.
- Las condiciones van en la sección `#Conditions`, no en `Event Grid.Load`.

Cada WebPanel con layout custom necesita un archivo companion `ObjectName.web.xml` en el mismo directorio. Usar el patrón `tabularGrid` de archivos existentes (ej. `src/Book/WorkWithWebBook/WWBook.web.xml`) como referencia.

## UI Architecture

- **Master page**: `MasterUnanimoSidebar` (módulo `General.UI`) — todas las páginas web la usan.
- **Sidebar navigation**: DataProvider `SidebarItemsDP` — agregar un bloque `SidebarItem` para registrar una nueva página en el sidebar.
- **Authorization**: Llamar `IsAuthorized(&PgmName)` en `Event Start`; redirigir con `NotAuthorized(&PgmName)` si es false.
- **WorkWith pattern**: Todos los CRUD se generan via el patrón WorkWith. Preferir extender el patrón antes de crear WebPanels standalone para gestión de entidades.

## Data Model (entidades core)

| Entidad | Atributos clave |
|---|---|
| Book | BookId, BookTitle, BookISBN, BookStatus (dominio ReadingStatus), BookRating (0–5), BookAcquisitionDate |
| Author | AuthorId, AuthorName, AuthorBirthDate |
| Loan | LoanId, BookId (FK), LoanPersonName, LoanDate, LoanExpectedReturnDate, LoanReturnedDate |
| Genre | GenreId, GenreName |
| Publisher | PublisherId, PublisherName |
| Shelf | ShelfId, ShelfName, ShelfDescription |

**Dominio ReadingStatus** — valores almacenados: `Pending`, `Reading`, `Read`, `Abandoned`.  
**Préstamo vencido**: `LoanReturnedDate` vacío + `LoanExpectedReturnDate < hoy`.

## Module references

- `GeneXus` core module: `ref/GeneXus/` — provee tipos base (`ObjectName`, `Url`, `Boolean`, etc.)
- `GeneXusUnanimo` v2.0.194: `ref/GeneXusUnanimo/` — design system, controles Chameleon, SDTs de sidebar, stencils.
- Referenciar objetos con su calificador de módulo cuando sea necesario: `'ObjectName, GeneXus'`, `'OverdueLoanItem, General.UI'`.
