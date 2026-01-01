# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RSS Reader is a native macOS application built with Swift that provides RSS feed management through both a menu bar interface and a desktop window. The app uses SwiftUI for the UI layer and SwiftData for persistence.

## Build & Run Commands

### Building
```bash
# Build the project
xcodebuild build -scheme RSSReader -project RSSReader.xcodeproj

# Open in Xcode
open RSSReader.xcodeproj
```

### Running
```bash
# From Xcode: Cmd + R
# From command line:
find ~/Library/Developer/Xcode/DerivedData -name "RSSReader.app" | head -n 1 | xargs open
```

### Testing
```bash
# Run tests
xcodebuild test -scheme RSSReader -project RSSReader.xcodeproj -destination 'platform=macOS'
```

## Architecture

### Dual UI System
The app operates in two modes simultaneously:
1. **Menu Bar Mode** (via `MenubarController`): Popover interface accessed from the status bar
2. **Desktop Mode** (via `RSSReaderApp`): Traditional window interface

Both modes share the same SwiftData `ModelContainer` and `ModelContext` for data consistency, but they are initialized separately in their respective components.

### Data Models (SwiftData)
Located in `RSSReader/Models/RSSmodel.swift`:

- **RSSFeedItem**: Individual RSS articles with title, link, pubDate, read status, and optional preview images
- **RSSFeedSource**: Feed sources with name, URL, and lastUpdated timestamp
- **DeletedArticle**: Archive of deleted article links to prevent re-adding previously deleted items
- **FilterOption**: Enum for filtering articles (all/unread/read/feed-specific)

### State Management Pattern
The app uses a centralized `ContentViewModel` (MVVM pattern) that:
- Owns the `ModelContext` reference
- Manages all CRUD operations for feeds and articles
- Handles filtering logic based on read status and feed selection
- Coordinates with `RSSParser` for feed fetching
- Implements "archive and delete" pattern: deleted items are recorded in `DeletedArticle` to prevent re-appearance

### RSS Parsing Architecture
`RSSParser` (`Services/RSSParser.swift`) handles:
- Asynchronous feed fetching with URLSession
- XML parsing with `XMLParserDelegate` pattern
- Support for both RSS and Atom formats
- Image extraction from multiple sources (enclosures, media:content, media:thumbnail, HTML in descriptions)
- Duplicate prevention using both existing items and deleted articles archive
- User notifications for new articles

### View Hierarchy
- **ContentView**: Root view with NavigationSplitView layout
- **SidebarView**: Left sidebar showing filters (All/Unread/Read) and feed list
- **MainContentView**: Main content area displaying article list
- **ArticleRow** / **RichArticleRow**: Two article display styles (configurable via `ArticleListStyle` enum)
- Modal sheets: AddFeedView, EditFeedView, ManageFeedsView

### Menu Bar Integration
`MenubarController` in `RSSReaderApp.swift`:
- Creates `NSStatusItem` with system icon
- Handles left-click (toggle popover) and right-click (show menu)
- Implements auto-refresh timer with configurable polling intervals (5/10/15/30 minutes)
- Uses `@AppStorage` for persistent settings (keepOpen, pollingInterval, showInMenuBar)
- Controls app activation policy (.regular vs .accessory) to toggle dock visibility

## Key Implementation Details

### Preventing Duplicate Articles
When parsing feeds, the system checks against:
1. Existing `RSSFeedItem` entries (by link)
2. `DeletedArticle` archive (by link)

This ensures that manually deleted articles won't reappear on subsequent refreshes.

### Shared ModelContext Pattern
Both `MenubarController` and `RSSReaderApp` create separate `ModelContainer` instances with the same schema. Changes made in one interface are visible in the other through SwiftData's persistence layer, though real-time synchronization between the two contexts is not explicitly implemented.

### Date Handling
RSS dates are stored as strings (`pubDate` field) with cleanup logic to remove timezone suffixes. Extensions in `String+Extensions.swift` provide date parsing utilities.

### Image Loading
The parser attempts to find preview images from multiple sources in order:
1. RSS enclosure elements (type="image/*")
2. Media namespace tags (media:content, media:thumbnail)
3. iTunes namespace (itunes:image)
4. Regex extraction from HTML in description/content fields

### Application Lifecycle
- The app doesn't terminate when the last window is closed (`applicationShouldTerminateAfterLastWindowClosed` returns `false`)
- Desktop window can be hidden via menu bar setting, switching between `.regular` and `.accessory` activation policies
- Default feed (joshwcomeau) is added on first launch if no feeds exist
