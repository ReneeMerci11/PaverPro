# PaverPro — Project Specification for Claude Code

## What This App Is
PaverPro is a native iOS app (Swift/SwiftUI) for contractors and skilled installers.
The user uploads a logo or design image + inputs their patio/driveway dimensions and
brick specs. PaverPro generates a mosaic visualization of that design inlaid in the
paver surface, a full material list (how many bricks of each color), and a cutting
guide (which bricks to cut, how, and where they go).

This is a contractor tool — outputs must be field-ready, precise, and printable.

## Patent Status
PATENT PENDING — provisional application filed with USPTO.
Core method (image → paver grid → color map → material count → cut instructions) is
protected. Do not add features that would expand scope without flagging for legal review.

---

## Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Platform | iOS native (SwiftUI) | Camera access, App Store, best UX for on-site use |
| Language | Swift 5.9+ | Native performance, vision frameworks |
| Image Processing | Apple Vision + Core Image | On-device, no API cost, fast |
| Color Matching | CIEDE2000 in Swift | Perceptually accurate color distance |
| Data Persistence | SwiftData | Simple local project storage |
| Export | PDFKit | Print-ready material lists and cut sheets |
| Minimum iOS | iOS 17 | SwiftData + latest Vision APIs |
| Architecture | MVVM | Clean separation, testable |

No backend required for V1. Everything runs on-device.

---

## App Flow (Screens in Order)

### 1. Home Screen
- List of saved projects
- "New Project" button
- Each project shows: name, thumbnail, date, brick count

### 2. New Project Setup
User inputs (in this order):
- Project name
- Installation width (ft/in) and height (ft/in)
- Primary brick dimensions (length x width, e.g. 4" x 8")
- Border brick dimensions (optional, different size)
- Mortar/joint width (default 0.25")

### 3. Design Image Upload
- Upload from photo library OR capture with camera
- Image is displayed with crop/rotate tool
- User confirms the design image

### 4. Color Palette Setup
V1: User manually picks 2–6 colors using a color picker
Each color gets:
- A name (e.g. "Charcoal", "White", "Tan")
- A color swatch (hex value)
V2: Pull from manufacturer catalog (Belgard, Unilock, Techo-Bloc)

### 5. Processing Screen
Show progress while the app:
1. Scales design image to paver grid dimensions
2. Reduces image to selected color palette (CIEDE2000 nearest neighbor)
3. Maps each grid cell to a brick color
4. Identifies partial bricks at boundaries
5. Calculates material counts
6. Generates cut instructions

### 6. Results Screen (3 tabs)

**Tab 1 — Mosaic Preview**
- Top-down grid visualization of the full design
- Each cell is filled with its mapped brick color
- Zoom and pan supported
- Color legend at bottom with brick counts

**Tab 2 — Material List**
For each color:
- Color swatch + name
- Full brick count (raw)
- Waste-adjusted count (+10% default, user can change %)
- Total across all colors at bottom
- Export as PDF button

**Tab 3 — Cut Guide**
List of every brick requiring a cut:
- Grid position (Row X, Col Y)
- Brick color
- Cut type (straight, angle, compound)
- Dimensions (e.g. "Cut 2.5\" from left edge")
- Small visual diagram of the cut
- Export as PDF button

### 7. Export / Share
- Full project PDF (preview + material list + cut guide)
- Share sheet (AirDrop, email, print)

---

## Core Data Models

```swift
// Project
struct PaverProject {
    var id: UUID
    var name: String
    var createdAt: Date
    var installationWidth: Double   // inches
    var installationHeight: Double  // inches
    var primaryBrick: BrickSpec
    var borderBrick: BrickSpec?
    var jointWidth: Double          // inches, default 0.25
    var colorPalette: [PaverColor]
    var designImageData: Data
    var grid: PaverGrid?            // nil until processed
}

// Brick specification
struct BrickSpec {
    var length: Double  // inches
    var width: Double   // inches
    var height: Double  // inches (not used in 2D but stored)
}

// A color in the palette
struct PaverColor {
    var id: UUID
    var name: String
    var hex: String     // e.g. "#4A4A4A"
    var rgb: (r: Int, g: Int, b: Int)
}

// The processed grid
struct PaverGrid {
    var columns: Int
    var rows: Int
    var cells: [[PaverCell]]    // [row][col]
}

// A single cell in the grid
struct PaverCell {
    var row: Int
    var col: Int
    var colorId: UUID
    var isFull: Bool
    var cutInstruction: CutInstruction?
}

// Cut instruction for a partial brick
struct CutInstruction {
    var cutType: CutType        // straight, angle, compound
    var measurements: [CutMeasurement]
    var diagramData: Data?      // rendered diagram image
}

enum CutType {
    case straight
    case angle
    case compound
}

struct CutMeasurement {
    var edge: BrickEdge         // top, bottom, left, right
    var distance: Double        // inches from that edge
    var angle: Double?          // degrees, nil for straight cuts
}

enum BrickEdge {
    case top, bottom, left, right
}
```

---

## Image Processing Logic

### Step 1: Grid Calculation
```
gridColumns = floor(installationWidth / (brickLength + jointWidth))
gridRows = floor(installationHeight / (brickWidth + jointWidth))
```

### Step 2: Image Scaling
Resize design image to exactly gridColumns x gridRows pixels.
Use Lanczos resampling for quality.

### Step 3: Color Reduction (CIEDE2000)
For each pixel in resized image:
1. Convert pixel RGB to LAB color space
2. For each color in palette, calculate CIEDE2000 distance
3. Assign pixel to palette color with minimum distance

### Step 4: Grid Population
Each pixel → one PaverCell
Cell gets the PaverColor matched in Step 3

### Step 5: Boundary / Cut Detection
A cell requires cutting if:
- It is at the edge of the installation area AND
- The design image has content (non-background) color there
  
OR if design region does not align to full brick boundaries
(detect via alpha channel or user-defined background color)

### Step 6: Cut Calculation
For each cell requiring a cut:
- Calculate what fraction of the brick is needed (e.g. 60% of length)
- Convert to physical measurement (fraction × brick length)
- Determine which edge(s) to cut from
- Generate CutInstruction with precise measurements

---

## Material Count Logic

```
For each PaverColor in palette:
  rawCount = number of cells assigned that color
  wasteAdjusted = ceil(rawCount * (1 + wasteFactorPercent / 100))

Total = sum of all wasteAdjusted counts
```

Default waste factor: 10%
User can adjust per color or globally in results screen.

---

## File Structure

```
PaverPro/
├── CLAUDE.md                      ← this file
├── PaverPro.xcodeproj
├── PaverPro/
│   ├── App/
│   │   ├── PaverProApp.swift
│   │   └── ContentView.swift
│   ├── Models/
│   │   ├── PaverProject.swift
│   │   ├── PaverGrid.swift
│   │   ├── PaverCell.swift
│   │   ├── PaverColor.swift
│   │   ├── BrickSpec.swift
│   │   └── CutInstruction.swift
│   ├── ViewModels/
│   │   ├── ProjectListViewModel.swift
│   │   ├── ProjectSetupViewModel.swift
│   │   ├── ProcessingViewModel.swift
│   │   └── ResultsViewModel.swift
│   ├── Views/
│   │   ├── Home/
│   │   │   ├── HomeView.swift
│   │   │   └── ProjectRowView.swift
│   │   ├── Setup/
│   │   │   ├── NewProjectView.swift
│   │   │   ├── DimensionInputView.swift
│   │   │   ├── DesignImageView.swift
│   │   │   └── ColorPaletteView.swift
│   │   ├── Processing/
│   │   │   └── ProcessingView.swift
│   │   └── Results/
│   │       ├── ResultsView.swift
│   │       ├── MosaicPreviewView.swift
│   │       ├── MaterialListView.swift
│   │       └── CutGuideView.swift
│   ├── Services/
│   │   ├── ImageProcessor.swift   ← core algorithm lives here
│   │   ├── ColorMatcher.swift     ← CIEDE2000 implementation
│   │   ├── GridGenerator.swift    ← grid calculation
│   │   ├── CutCalculator.swift    ← cut instruction generation
│   │   └── PDFExporter.swift      ← export logic
│   └── Utilities/
│       ├── ColorExtensions.swift
│       └── DoubleExtensions.swift
```

---

## Build Order (Phase by Phase)

### Phase 1 — Foundation (start here)
1. Xcode project setup with SwiftData
2. All data models defined
3. HomeView with mock data
4. NewProject flow (dimensions + brick specs input)

### Phase 2 — Core Algorithm
1. ColorMatcher.swift — CIEDE2000 implementation + tests
2. GridGenerator.swift — grid sizing from dimensions
3. ImageProcessor.swift — resize + color reduce + grid populate
4. CutCalculator.swift — boundary detection + cut math

### Phase 3 — Results UI
1. MosaicPreviewView — zoomable grid render
2. MaterialListView — counts + waste adjustment
3. CutGuideView — list with diagrams

### Phase 4 — Export
1. PDFExporter — full project PDF
2. Share sheet integration

### Phase 5 — Polish + App Store
1. App icon + launch screen
2. Onboarding flow
3. TestFlight beta
4. App Store submission

---

## Code Rules (always follow these)

- MVVM strictly — no business logic in Views
- All processing happens off main thread (async/await)
- No third-party dependencies in V1 — Apple frameworks only
- All measurements stored in inches internally, display in user's chosen unit
- Every function that touches the grid must have a unit test
- No force unwraps — use guard/let or if/let everywhere
- Accessibility: all interactive elements have accessibilityLabel

---

## What Claude Code Should NOT Do

- Do not add a backend or API calls in V1
- Do not use any third-party packages without flagging first
- Do not change the data models without updating this file
- Do not skip the processing off-main-thread requirement
- Do not store sensitive user data — projects are local only

---

## Current Status

[ ] Phase 1 — not started
[ ] Phase 2 — not started
[ ] Phase 3 — not started
[ ] Phase 4 — not started
[ ] Phase 5 — not started

**Next action: Run Phase 1, Step 1 — create Xcode project.**
