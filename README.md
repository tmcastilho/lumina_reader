# 📖 LUMINA READER — Digital eBook \& Document Ecosystem

**Single-File Client-Side eReader, Skeuomorphic Wooden Bookshelf, Multi-Format Document Engine (EPUB/PDF/TXT) \& Inline Versioned Editor**

!\[Lumina Reader Preview\](preview.jpg)

---

## 📑 Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Module 1: Wooden Bookshelf \& Library Manager (`#bookshelfView`)](#2-module-1-wooden-bookshelf--library-manager-bookshelfview)
3. [Module 2: Multi-Format Document Parsers (`EPUB` / `PDF` / `Reflow`)](#3-module-2-multi-format-document-parsers-epub--pdf--reflow)
4. [Module 3: Responsive Pagination, Paper Physics \& Reading Modes](#4-module-3-responsive-pagination-paper-physics--reading-modes)
5. [Module 4: Inline Page Editor \& Version Control (`v(1)`, `v(2)`)](#5-module-4-inline-page-editor--version-control-v1-v2)
6. [Client-Side Persistence \& State Matrix](#6-client-side-persistence--state-matrix)
7. [Resilience, Layout Isolation \& Best Practices](#7-resilience-layout-isolation--best-practices)
8. [License](#8-license)

---

## 1. Architecture Overview

The **Lumina Reader** is a zero-backend, single-file web application (`lumina_reader.html`) engineered for offline digital reading, annotation, and inline text correction. All file decompression, binary rendering, responsive column calculation, and storage persistence execute locally inside the browser runtime:

```
moon_reader_web.html (Single-File Architecture)
│
├── View 1: #bookshelfView (Primary Entry Point)
│   ├── Oak Wood Skeuomorphic UI (.wood-shelf-bg, .wooden-plank)
│   ├── Drag & Drop File Importer + Custom Shelf Reordering
│   └── Cover Aspect-Ratio Renderer (EPUB Cover / PDF Page 1 Thumbnail)
│
├── View 2: #readerViewportContainer (Secondary Reading & Editing View)
│   ├── #paginatedViewport  # CSS Multi-Column Responsive Reflow & Scroll Engine
│   ├── #pdfCanvasWrapper   # High-DPI Canvas Renderer (1-Page & 2-Page Spread)
│   ├── .paper-flip-leaf    # 3D Dual-Layer Paper Curl & Spine-Pivot Physics
│   └── #editModeBar        # ContentEditable Inline Corrector & v(n) Snapshotter
│
└── Client Storage Layer
    ├── IndexedDB (LuminaReaderDB -> books)  # Large EPUBs, PDF Uint8Arrays & Versions
    └── localStorage (lumina_*)             # Theme, Typography, Bookmarks & Highlights
```

### 1.1. Tech Stack

- **Core Runtime:** HTML5, Vanilla JavaScript (ES2023+), CSS3 Custom Properties (`[data-theme]`).
- **UI Styling \& Icons:** Tailwind CSS (CDN), Font Awesome `6.4.0`, Google Fonts (`Literata`, `Merriweather`, `Atkinson Hyperlegible`, `Roboto`).
- **Client-Side Document Engines:**
    - `JSZip 3.10.1` — In-memory ZIP decompression and XML/XHTML parsing for `.epub` archives.
    - `PDF.js 3.11.174` — High-resolution `<canvas>` page rendering and coordinate-grouped text extraction for `.pdf` files.
    - `KaTeX 0.16.9` — Auto-rendered inline (`$...$`) and display (`$$...$$`) LaTeX mathematical typesetting.

---

## 2. Module 1: Wooden Bookshelf \& Library Manager (`#bookshelfView`)

The bookshelf is strictly enforced as the primary landing screen (`#bookshelfView` visible on `DOMContentLoaded`), dynamically grouping books into horizontal oak shelves (`booksPerShelf` = `5` on desktop `>=1024px`, `4` on tablet `>=640px`, `3` on mobile) with bottom-aligned book spines (`items-end`).

### 2.1. Cover Extraction \& Aspect-Ratio Preservation

| Source Format | Cover Generation Strategy | Shelf Presentation |
| :--- | :--- | :--- |
| **EPUB** | Parses `META-INF/container.xml` $\\rightarrow$ `.opf` manifest (`meta[name="cover"]` or `properties="cover-image"`) and encodes the binary asset to a Base64 Data URL. | Clean full-bleed cover image preserving natural aspect ratio (`max-h-[210px]`) without overlaid text. |
| **PDF** | Renders page `1` via `pdfDoc.getPage(1)` at `scale: 0.6` onto an offscreen `<canvas>` and exports `image/jpeg` (`0.8` quality). | Clean document thumbnail resting directly on the wooden plank bevel. |
| **TXT / Manual** | Generates a typographic hardcover card with customizable gradient backgrounds and spine crease overlays. | Classic `2:3` aspect-ratio book cover with title, author, and format badge. |

### 2.2. Interactive Drag \& Drop Reordering (`custom` Sort)

Bookshelf cards implement native HTML5 Drag \& Drop (`draggable="true"`, `dragstart`, `dragover`, `drop`) isolated from the external file drop-zone (`#dropZoneOverlay`) via `state.draggedBookId`:

```
[ Card Drag Start ] ──► [ Drop on Target Book Card ]
                                │
                                ▼
              reorderBooksOnShelf(sourceId, targetId)
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
   Updates customOrder index         Switches #shelfSortSelect
   & persists to IndexedDB           to "custom" (Personalizada)
```

---

## 3. Module 2: Multi-Format Document Parsers (`EPUB` / `PDF` / `Reflow`)

### 3.1. EPUB Pipeline (`parseEpubFile`)

1. **Container \& OPF Resolution:** Reads `META-INF/container.xml` to locate the root `package.opf` path and base directory (`opfDir`).
2. **Metadata \& TOC Mapping:** Extracts Dublin Core `<dc:title>` and `<dc:creator>`, then parses `toc.ncx` (`navPoint` $\\rightarrow$ `navLabel` / `content[src]`) into `tocMap` to label spine chapters accurately.
3. **Relative Path \& Asset Inlining:** Resolves relative paths across nested directories (`resolveZipPath`) and converts all internal `<img>` and SVG `<image>` references into cached Base64 Data URLs (`imageCache`) so books render 100% offline.

### 3.2. Dual-Mode PDF Engine (`parsePdfFile` \& `renderPdfPageOnCanvas`)

PDF documents support real-time switching (`#btnTogglePdfMode`) between two distinct rendering pipelines:

- **Original Canvas Mode (`pdfRenderMode = 'canvas'`):**
    - Calculates `fitScale = Math.min(maxAvailWidth / unscaledViewport.width, maxAvailHeight / unscaledViewport.height)` so pages fit 100% inside the screen without vertical clipping or aspect distortion.
    - Renders at `fitScale * Math.max(1.5, window.devicePixelRatio)` for sharp text on Retina/OLED screens, dynamically resizing `#pageSheet` to match the exact document dimensions.
    - Applies theme-aware CSS filters (`.pdf-canvas-themed`) for **Sepia**, **OLED Dark**, and **Warm Night** modes.
- **Text Reflow Mode (`pdfRenderMode = 'reflow'`):**
    - Groups `page.getTextContent()` items by vertical transform coordinates (`Math.abs(y - lastY) > 14`) into semantic `<p>` blocks, unlocking font resizing, family switching, and text highlighting.

---

## 4. Module 3: Responsive Pagination, Paper Physics \& Reading Modes

### 4.1. Real-Time Multi-Column Pagination (`rebuildResponsivePages`)

Instead of truncating overflow text with vertical scrollbars, the paginated engine measures the exact inner dimensions of `#paginatedViewport` and injects CSS `column-width` and `column-gap` properties:

- **Zero Edge Bleed:** In single-page mode, `columnGap` is enforced as `Math.max(80, Math.ceil(padLeft + padRight + 32))`, ensuring adjacent columns never peek into the right margin.
- **Automatic Resize Recalculation:** Window resize or font-size changes preserve the user's exact reading progress ratio (`state.currentPage / (state.totalPages - 1)`) while rebuilding `state.flatPages`.

### 4.2. Reading Modes \& Hardware Input Matrix

| Mode / Input | Behavior \& Physics |
| :--- | :--- |
| **1-Page Paginated** | Full-sheet `rotateY(-105deg)` paper curl from right-to-left when advancing (`paperTurnForward`) and left-to-right when returning (`paperTurnBackward`). |
| **2-Page Open Book Spread** | Activated automatically on viewports `>= 900px` (or manually via `#btnToggleSpread`). Uses a central spine crease (`.book-center-crease`) and pivots `.paper-flip-leaf` strictly around the center axis (`left: 50%` when advancing right-to-left; `right: 50%` when returning left-to-right). |
| **Infinite Scroll (`scroll`)** | Locks layout to a single continuous vertical column (`#paginatedViewport.scroll-reading-mode` or `#pdfInfiniteScrollWrapper`) and disables the 2-page spread toggle. |
| **Mouse Wheel Hardware (`wheel`)** | In paginated mode, vertical wheel scroll triggers `turnPage(1)` or `turnPage(-1)` with a `340 ms` debounce guard while ignoring horizontal scrolls inside `.katex-display` equations. |

---

## 5. Module 4: Inline Page Editor \& Version Control (`v(1)`, `v(2)`)

The inline editor (`enterTextEditMode` / `saveEditedBookVersion`) allows users to fix typos, spacing errors, or archaic spelling directly on the rendered page without mutating the original book:

```
[ Click "Editar" ] ──► Disables tap overlays (#tapLeft, #tapCenter, #tapRight)
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
    Single Chapter Active         2-Page Split Chapters
 (contenteditable on article)   (#editLeftChapterBox + #editRightChapterBox)
              │                             │
              └──────────────┬──────────────┘
                             ▼
                [ Click "Salvar v(n)" ]
                             │
                             ▼
   Clones chapters array ──► Appends incremental suffix v(1), v(2)...
                             ──► Saves new record to IndexedDB
                             ──► Switches active reader to new version
```

- **Cancel Action (`exitTextEditMode`):** Restores `state.editOriginalHtml`, cleans up `contenteditable` attributes, and returns directly to `#bookshelfView`.

---

## 6. Client-Side Persistence \& State Matrix

### 6.1. IndexedDB Schema (`LuminaReaderDB`)

| Object Store | Key Path | Stored Payload Properties |
| :--- | :--- | :--- |
| `books` | `id` | `id`, `title`, `author`, `coverBg`, `coverImage` (Base64), `badge`, `format` (`epub` \| `pdf` \| `html`), `pdfData` (`Uint8Array`), `lastReadPage`, `totalPages`, `customOrder`, `dateAdded`, `chapters[]` |

### 6.2. `localStorage` Keys

| Key | Type | Purpose |
| :--- | :--- | :--- |
| `lumina_theme` | `string` | Active color theme (`sepia`, `light`, `dark`, `warm`). |
| `lumina_fontsize` | `number` | Reader font size in pixels (`14`–`32`). |
| `lumina_fontfamily` | `string` | Active CSS font family stack. |
| `lumina_spread` | `string` | Spread layout preference (`auto`, `single`, `double`). |
| `lumina_reading_flow` | `string` | Flow mode (`paginated` or `scroll`). |
| `lumina_shelf_sort` | `string` | Shelf sorting mode (`recent`, `custom`, `progress`, `title`). |
| `lumina_bookmarks` | `JSON Array` | Saved page bookmarks (`bookId`, `page`, `chapter`, `snippet`, `date`). |
| `lumina_highlights` | `JSON Array` | Text highlights (`bookId`, `page`, `text`, `color`, `date`). |

---

## 7. Resilience, Layout Isolation \& Best Practices

- **IndexedDB Over** **`localStorage`** **for Books:** Binary PDF buffers (`Uint8Array`) and multi-megabyte EPUB image assets are stored exclusively in `IndexedDB` to avoid the `5 MB` `QuotaExceededError` limit of `localStorage`.
- **KaTeX Formula Overflow Protection:** Block equations (`.katex-display`) use `overflow-x: auto` and `white-space: nowrap` inside `.katex` so complex formulas never break multi-column pagination boundaries.
- **Custom Thin Scrollbars:** Global thin scrollbars (`5px`, `rgba(54, 54, 62, 0.78)`) with zero directional buttons ensure high contrast across both light/sepia themes and `#121212` OLED dark backgrounds.
- **Drag-State Collision Guard:** Distinguishes internal shelf reordering (`state.draggedBookId !== null`) from OS file drag-and-drop (`DataTransfer.files`), preventing `#dropZoneOverlay` from intercepting shelf card rearrangements.

---

## 8. License

Copyright (c) 2026 **tmcastilho**. All rights reserved.

Copying, modification, distribution, sublicensing, or any use of this software and associated files, in whole or in part, for any purpose, is strictly prohibited without prior written permission from the copyright holder.
