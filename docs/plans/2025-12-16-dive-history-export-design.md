# Dive History Export and Visualization - Design Document

**Date:** 2025-12-16
**Status:** Draft for Review
**Goal:** Enable users to download dive logs from Cosmiq 5 and visualize them directly in the browser

## 1. Overview

This feature adds dive history export and visualization to the Cosmiq 5 web manager. Users will be able to download their dive logs via Bluetooth and immediately view depth/time profiles rendered as inline SVG, with no external dependencies or Python scripts required.

### Success Criteria
- **Minimum Viable:** Users can download dive logs and view depth/time profile graphs in the browser
- **Ideal:** Export to standard dive log formats (UDDF, Subsurface) for compatibility with other software

### Design Constraints
- Zero external dependencies (no Chart.js, no matplotlib)
- Pure HTML5 SVG + vanilla JavaScript (no Canvas)
- All processing in-memory (no file downloads required)
- Consistent with existing codebase style (single HTML file)

### Why SVG Over Canvas?
- **Declarative:** Easier to debug and inspect in DevTools
- **Scalable:** Crisp at any resolution, responsive by default
- **CSS styleable:** Can use existing CSS variables and animations
- **Interactive:** Easy to add tooltips and hover effects
- **Exportable:** Can save as .svg file for use in reports
- **Accessible:** Screen readers can parse SVG text elements

## 2. Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 1: DATA ACQUISITION                    │
│                  (Export from Device → Memory)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User clicks "Download Logs"                                    │
│         ↓                                                       │
│  Send BLE commands (#41 header, #43 body)                       │
│         ↓                                                       │
│  Device streams packets (0x42 header, 0x44 body)                │
│         ↓                                                       │
│  Collect packets in memory (hex strings)                        │
│         ↓                                                       │
│  Parse headers (72 bytes → dive metadata objects)               │
│         ↓                                                       │
│  Parse body (binary → depth sample arrays)                      │
│         ↓                                                       │
│  Store in diveDatabase (in-memory cache)                        │
│                                                                 │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      │ Well-defined data structure
                      │ (Array of dive objects)
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│                   PHASE 2: DATA CONSUMPTION                     │
│                  (Visualize / Export / Analyze)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ VISUALIZATION (MVP)                                 │       │
│  │  • Show dive list                                   │       │
│  │  • Render SVG depth/time graphs                     │       │
│  │  • Interactive tooltips (Phase 2)                   │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ EXPORT (Future)                                     │       │
│  │  • UDDF format (Subsurface compatible)              │       │
│  │  • CSV for spreadsheets                             │       │
│  │  • JSON raw data backup                             │       │
│  │  • SVG graph download                               │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ ANALYTICS (Future)                                  │       │
│  │  • Dive statistics                                  │       │
│  │  • Multi-dive comparison                            │       │
│  │  • Safety analysis                                  │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key Design Principle:** Phase 1 and Phase 2 are completely decoupled. The `diveDatabase` in-memory storage is the single interface between them. This makes it trivial to add new export formats or visualization types without touching the BLE/parsing code.

### Component Breakdown

The architecture is split into **two distinct phases** for clean separation of concerns:

---

### **PHASE 1: Data Acquisition** (Export from device → memory)

This phase is responsible for getting dive data from the device and storing it as structured JavaScript objects.

**A. BLE Download Module** (exists in `divelog_download_parse` branch)
- Manages packet collection state
- Sends header/body request commands (`#41`, `#43`)
- Accumulates incoming hex strings from device responses (`0x42`, `0x44`)
- Shows download progress
- **Output:** Raw hex strings (header + body)

**B. Binary Parser Module** (new - port from Python)
- `parseHeaders(hexString)` → Array of `DiveHeader` objects
- `parseSamples(bodyHex, header)` → Array of `DiveSample` objects
- Handle little-endian byte unpacking in JavaScript
- **Output:** Structured dive data in memory

**C. Data Storage Module** (new)
- `diveDatabase` - In-memory storage for parsed dives
- `cacheDive(header, samples)` → Store a single dive
- `getAllDives()` → Retrieve all cached dives
- `getDive(logNumber)` → Retrieve specific dive
- `clearCache()` → Reset storage
- **Purpose:** Single source of truth for dive data during session

**Key principle:** Phase 1 outputs a well-defined data structure that Phase 2 can consume in any way.

---

### **PHASE 2: Data Consumption** (Visualize / Export)

This phase decides **what to do** with the dive data - display it, export it, analyze it, etc. All modules in this phase are **independent and extensible**.

**D. Visualization Module** (new)
- `renderDiveList(dives)` → Show list of available dives
- `renderDiveProfile(dive, containerId)` → Generate SVG visualization
- Generate SVG elements: axes, grid lines, labels
- Plot depth samples as `<polyline>` or `<path>`
- Style with CSS variables from existing theme
- **Future:** Could add different chart types (3D, comparative, heat maps, etc.)

**E. Export Modules** (future enhancement)
- `exportToUDDF(dives)` → Universal Dive Data Format XML
- `exportToSubsurface(dives)` → Subsurface XML dialect
- `exportToCSV(dives)` → Spreadsheet-friendly format
- `exportToJSON(dives)` → Raw data backup
- `exportCurrentSVG()` → Save displayed graph as .svg file
- **Future:** Could add Garmin, Shearwater, MacDive formats, etc.

**F. Analytics Module** (future enhancement)
- Calculate dive statistics (avg depth, max time, etc.)
- Compare dives over time
- Generate reports

---

**Benefits of this separation:**
1. **Testable:** Can test parsing independently from visualization
2. **Reusable:** Same data can feed multiple outputs (SVG + UDDF + CSV)
3. **Extensible:** Easy to add new export formats or visualizations
4. **Maintainable:** Changes to visualization don't affect data acquisition
5. **Cacheable:** Once downloaded, data persists in memory for the session

## 3. Data Structures

**These data structures form the contract between Phase 1 (acquisition) and Phase 2 (consumption).**

### In-Memory Storage

```javascript
// Global storage for parsed dive data
const diveDatabase = {
    dives: [],           // Array of complete dive objects
    rawBodyHex: '',      // Cached body hex for re-parsing if needed
    lastDownload: null   // Timestamp of last download
};

// Complete dive object combining header + samples
const completeDive = {
    header: DiveHeader,     // Metadata (see below)
    samples: [DiveSample],  // Array of depth/time samples (see below)
    parsed: Date.now()      // When this was parsed
};
```

### Dive Header Object
```javascript
{
    logNumber: 1,              // Dive number
    mode: 0,                   // 0=Scuba, 1=Gauge, 2=Freedive
    oxygenPercent: 21,         // Air mix
    date: {
        year: 2025,
        month: 12,
        day: 16,
        hour: 14,
        minute: 30
    },
    durationMinutes: 45,       // Total dive time
    maxDepthMeters: 18.5,      // Maximum depth
    minTempCelsius: 12.3,      // Minimum temperature
    logPeriod: 1,              // Sampling interval in seconds
    logLength: 2700,           // Body data length in bytes
    startSector: 12,           // Memory sector location
    endSector: 13
}
```

### Dive Sample Object
```javascript
{
    timeSeconds: 0,            // Time since dive start
    depthMeters: 18.5,         // Depth at this sample
    marker: 0xC2               // Raw marker byte (for debugging)
}
```

## 4. Implementation Details

### Part 1: BLE Download (Reuse Existing)

The `divelog_download_parse` branch already has working code:

```javascript
// State management
const dumpState = {
    active: false,
    phase: 0,      // 0=header, 1=body
    packets: []    // Collected hex strings
};

// Trigger download
async function downloadLogs() {
    dumpState.active = true;
    dumpState.packets = [];

    // Request header
    await sendStatic("#41BD0200", "Request Header");

    // Wait for header response, then request body
    // (handled in handleRX when 0x42 packets arrive)
}

// In handleRX(), collect packets:
if (cmd === "42") {
    dumpState.packets.push(line);
    // When complete, trigger body request
} else if (cmd === "44") {
    dumpState.packets.push(line);
    // Accumulate body packets
}
```

**Key modification:** Instead of saving to file, call parser when download completes.

### Part 2: Binary Parsing (Port from Python)

**Parse Headers:**
```javascript
function parseHeaders(headerHex) {
    // headerHex is concatenated payload from all 0x42 packets
    const bytes = hexToBytes(headerHex);
    const dives = [];

    // Each header is 72 bytes
    for (let i = 0; i < bytes.length; i += 72) {
        if (i + 72 > bytes.length) break;

        const chunk = bytes.slice(i, i + 72);

        // Skip empty headers
        const logNum = readUint16LE(chunk, 0);
        if (logNum === 0 || logNum === 0xFFFF) continue;

        const dive = {
            logNumber: logNum,
            mode: chunk[2],
            oxygenPercent: chunk[3],
            date: {
                year: readUint16LE(chunk, 6) + 2000,
                month: (readUint16LE(chunk, 8) >> 8) & 0xFF,
                day: readUint16LE(chunk, 8) & 0xFF,
                hour: (readUint16LE(chunk, 10) >> 8) & 0xFF,
                minute: readUint16LE(chunk, 10) & 0xFF
            },
            durationMinutes: readUint16LE(chunk, 12),
            maxDepthMeters: readUint16LE(chunk, 22) / 100.0,
            minTempCelsius: readInt16LE(chunk, 24) / 10.0,
            logPeriod: chunk[28],
            logLength: readUint16LE(chunk, 28),
            startSector: readUint16LE(chunk, 30),
            endSector: readUint16LE(chunk, 32)
        };

        dives.push(dive);
    }

    return dives;
}
```

**Parse Body Samples:**
```javascript
function parseSamples(bodyHex, header) {
    const bytes = hexToBytes(bodyHex);
    const SECTOR_SIZE = 4096;

    // Calculate offset for this dive's data
    const startOffset = (header.startSector - 12) * SECTOR_SIZE;
    const diveData = bytes.slice(startOffset, startOffset + header.logLength);

    const samples = [];
    const validMarkers = [
        0xC0, 0xC1, 0xC2, 0xC3, 0xC4, 0xC5, 0xC6, 0xC7,
        0xC8, 0xC9, 0xCA, 0xCB, 0xCC, 0xCD, 0xCE, 0xCF,
        0xB0, 0xB1, 0xB2, 0xB3, 0xB4, 0xB5, 0xB6, 0xB7,
        0xB8, 0xB9, 0xBA, 0xBB, 0xBC, 0xBD, 0xBE, 0xBF
    ];

    let i = 0;
    let sampleIndex = 0;

    // Parse 4-byte samples: [Marker] [0x00] [Depth_Low] [Depth_High]
    while (i < diveData.length - 4) {
        const marker = diveData[i];

        if (diveData[i + 1] === 0x00 && validMarkers.includes(marker)) {
            const depthRaw = readUint16LE(diveData, i + 2);

            // Sanity check: max dive computer depth ~200m
            if (depthRaw > 0 && depthRaw < 20000) {
                samples.push({
                    timeSeconds: sampleIndex * header.logPeriod,
                    depthMeters: depthRaw / 100.0,
                    marker: marker
                });
                sampleIndex++;
            }

            i += 4;  // Move to next sample
        } else {
            i += 1;  // Scan forward
        }
    }

    return samples;
}
```

**Helper Functions:**
```javascript
function hexToBytes(hexString) {
    const bytes = [];
    for (let i = 0; i < hexString.length; i += 2) {
        bytes.push(parseInt(hexString.substr(i, 2), 16));
    }
    return bytes;
}

function readUint16LE(bytes, offset) {
    return bytes[offset] | (bytes[offset + 1] << 8);
}

function readInt16LE(bytes, offset) {
    const val = readUint16LE(bytes, offset);
    return val > 32767 ? val - 65536 : val;
}
```

### Part 3: SVG Rendering

**Render Dive Profile:**
```javascript
function renderDiveProfile(containerId, samples, metadata) {
    const container = document.getElementById(containerId);

    // SVG dimensions
    const width = 800;
    const height = 500;
    const padding = 60;
    const graphWidth = width - 2 * padding;
    const graphHeight = height - 2 * padding;

    // Find data bounds
    const maxDepth = Math.max(...samples.map(s => s.depthMeters));
    const maxTime = Math.max(...samples.map(s => s.timeSeconds));

    // Start building SVG
    let svg = `<svg width="${width}" height="${height}" xmlns="http://www.w3.org/2000/svg" style="background: var(--bg); font-family: var(--mono);">`;

    // Title
    const title = `Dive #${metadata.logNumber} - ${metadata.date.year}/${metadata.date.month}/${metadata.date.day} ${metadata.date.hour}:${String(metadata.date.minute).padStart(2, '0')}`;
    svg += `<text x="${width / 2}" y="30" text-anchor="middle" font-size="16" font-weight="bold" fill="var(--text)">${title}</text>`;

    // Draw grid - horizontal lines (depth)
    const depthStep = Math.ceil(maxDepth / 5);
    for (let d = 0; d <= maxDepth; d += depthStep) {
        const y = padding + (d / maxDepth) * graphHeight;

        // Grid line
        svg += `<line x1="${padding}" y1="${y}" x2="${padding + graphWidth}" y2="${y}" stroke="#e0e0e0" stroke-width="1"/>`;

        // Label
        svg += `<text x="${padding - 10}" y="${y + 4}" text-anchor="end" font-size="12" fill="#666">${d}m</text>`;
    }

    // Draw grid - vertical lines (time)
    const timeStep = Math.ceil(maxTime / 10 / 60) * 60; // Round to minutes
    for (let t = 0; t <= maxTime; t += timeStep) {
        const x = padding + (t / maxTime) * graphWidth;

        // Grid line
        svg += `<line x1="${x}" y1="${padding}" x2="${x}" y2="${padding + graphHeight}" stroke="#e0e0e0" stroke-width="1"/>`;

        // Label
        svg += `<text x="${x}" y="${height - padding + 20}" text-anchor="middle" font-size="12" fill="#666">${Math.floor(t / 60)}min</text>`;
    }

    // Draw axes border
    svg += `<rect x="${padding}" y="${padding}" width="${graphWidth}" height="${graphHeight}" fill="none" stroke="#333" stroke-width="2"/>`;

    // Plot dive profile as polyline
    const points = samples.map(s => {
        const x = padding + (s.timeSeconds / maxTime) * graphWidth;
        const y = padding + (s.depthMeters / maxDepth) * graphHeight;
        return `${x},${y}`;
    }).join(' ');

    svg += `<polyline points="${points}" fill="none" stroke="var(--blue)" stroke-width="2"/>`;

    // Add area fill for visual appeal (optional)
    const areaPoints = `${padding},${padding} ${points} ${padding + graphWidth},${padding}`;
    svg += `<polygon points="${areaPoints}" fill="var(--blue)" opacity="0.1"/>`;

    // Draw stats at bottom
    svg += `<text x="${padding}" y="${height - 10}" font-size="12" fill="var(--text)">Max Depth: ${metadata.maxDepthMeters.toFixed(1)}m</text>`;
    svg += `<text x="${padding + 200}" y="${height - 10}" font-size="12" fill="var(--text)">Duration: ${metadata.durationMinutes}min</text>`;
    svg += `<text x="${padding + 400}" y="${height - 10}" font-size="12" fill="var(--text)">Min Temp: ${metadata.minTempCelsius.toFixed(1)}°C</text>`;

    svg += '</svg>';

    // Insert SVG into container
    container.innerHTML = svg;
}
```

**Alternative: Generate SVG DOM Elements (More Interactive)**

For better interactivity (tooltips, hover effects), you can create actual DOM elements instead of string concatenation:

```javascript
function renderDiveProfileDOM(containerId, samples, metadata) {
    const container = document.getElementById(containerId);
    container.innerHTML = ''; // Clear previous content

    // Create SVG element
    const svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
    svg.setAttribute('width', 800);
    svg.setAttribute('height', 500);
    svg.style.background = 'var(--bg)';
    svg.style.fontFamily = 'var(--mono)';

    const width = 800;
    const height = 500;
    const padding = 60;
    const graphWidth = width - 2 * padding;
    const graphHeight = height - 2 * padding;

    const maxDepth = Math.max(...samples.map(s => s.depthMeters));
    const maxTime = Math.max(...samples.map(s => s.timeSeconds));

    // Helper to create SVG elements
    function createSVGElement(type, attrs) {
        const el = document.createElementNS('http://www.w3.org/2000/svg', type);
        Object.entries(attrs).forEach(([key, value]) => {
            el.setAttribute(key, value);
        });
        return el;
    }

    // Title
    const title = createSVGElement('text', {
        x: width / 2,
        y: 30,
        'text-anchor': 'middle',
        'font-size': 16,
        'font-weight': 'bold',
        fill: 'var(--text)'
    });
    title.textContent = `Dive #${metadata.logNumber} - ${metadata.date.year}/${metadata.date.month}/${metadata.date.day}`;
    svg.appendChild(title);

    // Grid and axes (similar to string version)
    // ... [grid code omitted for brevity]

    // Plot polyline
    const points = samples.map(s => {
        const x = padding + (s.timeSeconds / maxTime) * graphWidth;
        const y = padding + (s.depthMeters / maxDepth) * graphHeight;
        return `${x},${y}`;
    }).join(' ');

    const polyline = createSVGElement('polyline', {
        points: points,
        fill: 'none',
        stroke: 'var(--blue)',
        'stroke-width': 2
    });
    svg.appendChild(polyline);

    // Add interactive circles at data points (optional)
    samples.forEach(s => {
        const x = padding + (s.timeSeconds / maxTime) * graphWidth;
        const y = padding + (s.depthMeters / maxDepth) * graphHeight;

        const circle = createSVGElement('circle', {
            cx: x,
            cy: y,
            r: 3,
            fill: 'var(--blue)',
            opacity: 0
        });

        // Add tooltip on hover
        circle.addEventListener('mouseenter', (e) => {
            circle.setAttribute('opacity', 1);
            circle.setAttribute('r', 5);
            // Show tooltip with depth and time
            const tooltip = document.getElementById('diveTooltip');
            tooltip.innerText = `${s.depthMeters.toFixed(1)}m @ ${Math.floor(s.timeSeconds / 60)}:${String(s.timeSeconds % 60).padStart(2, '0')}`;
            tooltip.style.display = 'block';
            tooltip.style.left = e.pageX + 'px';
            tooltip.style.top = e.pageY + 'px';
        });

        circle.addEventListener('mouseleave', () => {
            circle.setAttribute('opacity', 0);
            circle.setAttribute('r', 3);
            document.getElementById('diveTooltip').style.display = 'none';
        });

        svg.appendChild(circle);
    });

    container.appendChild(svg);
}
```

### Part 4: UI Integration

**Add to Diagnostics Tab:**
```html
<div id="diag" class="section">
    <!-- Existing byte hunter code -->

    <!-- New dive log section -->
    <div class="card">
        <h3>Dive Log Download</h3>
        <button class="btn-main" onclick="downloadDiveLogs()">
            Download All Dive Logs
        </button>
        <div id="downloadProgress" style="display:none;">
            <p id="downloadStatus">Downloading...</p>
            <div style="width:100%; background:#ddd; height:20px; border-radius:10px;">
                <div id="downloadBar" style="width:0%; background:var(--blue); height:100%; border-radius:10px;"></div>
            </div>
        </div>
    </div>

    <div id="diveList" style="display:none;">
        <h3>Downloaded Dives</h3>
        <div id="diveCards"></div>
    </div>

    <!-- SVG container for dive profile -->
    <div id="diveGraphContainer" style="display:none; margin-top:20px;"></div>

    <!-- Tooltip for interactive hover (optional) -->
    <div id="diveTooltip" style="display:none; position:absolute; background:#333; color:white; padding:5px 10px; border-radius:5px; font-size:12px; pointer-events:none; z-index:1000;"></div>
</div>
```

**Orchestration Functions:**

```javascript
// =============================================================================
// PHASE 1: DATA ACQUISITION
// =============================================================================

/**
 * Initiates BLE download of all dive logs from device.
 * This is the entry point for Phase 1.
 */
async function downloadDiveLogs() {
    // Show progress UI
    document.getElementById('downloadProgress').style.display = 'block';
    document.getElementById('downloadStatus').innerText = 'Requesting header...';

    // Clear state
    dumpState.active = true;
    dumpState.phase = 0;
    dumpState.packets = [];

    // Request header (metadata about all dives)
    await sendStatic("#41BD0200", "Request Header");

    // Body request happens automatically in handleRX when header completes
    // When all packets collected, handleRX calls processDiveLogs()
}

/**
 * Processes downloaded hex data and stores it in memory.
 * This completes Phase 1 and transitions to Phase 2.
 */
function processDiveLogs() {
    document.getElementById('downloadStatus').innerText = 'Processing...';

    // Separate header and body packets from raw BLE stream
    const headerHex = dumpState.packets
        .filter(p => p.startsWith('42'))
        .map(p => p.substring(6))  // Strip command/checksum/length
        .join('');

    const bodyHex = dumpState.packets
        .filter(p => p.startsWith('44'))
        .map(p => p.substring(6))
        .join('');

    // Parse headers to get dive metadata
    const headers = parseHeaders(headerHex);

    // Store in memory for Phase 2 consumption
    diveDatabase.dives = headers.map(header => ({
        header: header,
        samples: null  // Lazy-load samples when needed
    }));
    diveDatabase.rawBodyHex = bodyHex;
    diveDatabase.lastDownload = Date.now();

    console.log(`Phase 1 complete: ${diveDatabase.dives.length} dives stored in memory`);

    // Hide download UI
    document.getElementById('downloadProgress').style.display = 'none';

    // Transition to Phase 2: Show dive list
    showDiveList();
}

// =============================================================================
// PHASE 2: DATA CONSUMPTION - VISUALIZATION
// =============================================================================

/**
 * Displays list of available dives from memory.
 * Consumes data from Phase 1 (diveDatabase).
 */
function showDiveList() {
    const dives = diveDatabase.dives;

    if (dives.length === 0) {
        log('No dives found in memory. Download first.', 'warn');
        return;
    }

    const diveCards = document.getElementById('diveCards');
    diveCards.innerHTML = '';

    dives.forEach((dive, index) => {
        const h = dive.header;
        const card = document.createElement('div');
        card.className = 'card';
        card.style.cursor = 'pointer';
        card.innerHTML = `
            <h4>Dive #${h.logNumber}</h4>
            <p>${h.date.year}/${h.date.month}/${h.date.day} ${h.date.hour}:${String(h.date.minute).padStart(2, '0')}</p>
            <p>Max Depth: ${h.maxDepthMeters.toFixed(1)}m | Duration: ${h.durationMinutes}min</p>
        `;
        card.onclick = () => viewDive(index);
        diveCards.appendChild(card);
    });

    document.getElementById('diveList').style.display = 'block';
}

/**
 * Renders SVG visualization for a specific dive.
 * Lazy-loads samples from cached body hex if not already parsed.
 */
function viewDive(diveIndex) {
    const dive = diveDatabase.dives[diveIndex];

    // Lazy-load samples if not already parsed
    if (!dive.samples) {
        dive.samples = parseSamples(diveDatabase.rawBodyHex, dive.header);
        console.log(`Parsed ${dive.samples.length} samples for dive #${dive.header.logNumber}`);
    }

    // Render SVG
    const container = document.getElementById('diveGraphContainer');
    container.style.display = 'block';
    renderDiveProfile('diveGraphContainer', dive.samples, dive.header);

    // Scroll to graph
    container.scrollIntoView({ behavior: 'smooth' });
}

// =============================================================================
// PHASE 2: DATA CONSUMPTION - EXPORT (FUTURE)
// =============================================================================

/**
 * Exports all dives to UDDF format.
 * Consumes data from Phase 1 (diveDatabase).
 */
function exportToUDDF() {
    const dives = diveDatabase.dives;

    // Ensure all samples are loaded
    dives.forEach(dive => {
        if (!dive.samples) {
            dive.samples = parseSamples(diveDatabase.rawBodyHex, dive.header);
        }
    });

    // Generate UDDF XML (implementation in Section 5)
    const xml = generateUDDFXML(dives);

    // Trigger download
    downloadFile('cosmiq5_dives.uddf', xml, 'application/xml');
}

/**
 * Helper: Trigger browser download of generated file.
 */
function downloadFile(filename, content, mimeType) {
    const blob = new Blob([content], { type: mimeType });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
}
```

## 5. Future Enhancements (Post-MVP)

### Export to UDDF Format
Universal Dive Data Format is an XML standard supported by many dive apps:

```javascript
function exportToUDDF(dives, bodyHex) {
    const xml = `<?xml version="1.0" encoding="UTF-8"?>
<uddf version="3.2.0">
    <generator>
        <name>Cosmiq 5 Web Manager</name>
        <version>1.0</version>
    </generator>
    <diver>
        <owner id="owner1">
            <equipment>
                <divecomputer id="cosmiq5">
                    <name>Deepblu Cosmiq 5</name>
                    <model>Cosmiq 5</model>
                </divecomputer>
            </equipment>
        </owner>
    </diver>
    <profiledata>
        ${dives.map(dive => {
            const samples = parseSamples(bodyHex, dive);
            return `
        <repetitiongroup id="rg${dive.logNumber}">
            <dive id="dive${dive.logNumber}">
                <informationbeforedive>
                    <datetime>${formatUDDFDate(dive.date)}</datetime>
                    <divenumber>${dive.logNumber}</divenumber>
                </informationbeforedive>
                <samples>
                    ${samples.map(s => `
                    <waypoint>
                        <divetime>${s.timeSeconds}</divetime>
                        <depth>${s.depthMeters}</depth>
                    </waypoint>`).join('')}
                </samples>
                <informationafterdive>
                    <greatestdepth>${dive.maxDepthMeters}</greatestdepth>
                    <diveduration>${dive.durationMinutes * 60}</diveduration>
                    <lowesttemperature>${dive.minTempCelsius}</lowesttemperature>
                </informationafterdive>
            </dive>
        </repetitiongroup>`;
        }).join('')}
    </profiledata>
</uddf>`;

    // Trigger download
    const blob = new Blob([xml], { type: 'application/xml' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `cosmiq5_dives_${Date.now()}.uddf`;
    a.click();
}
```

### Export to Subsurface Format
Subsurface uses a custom XML dialect. Similar approach to UDDF but different schema.

## 6. Testing Strategy

### Unit Tests (Manual)
- **Parser validation:** Use known hex dumps from branch to verify parsing
- **Boundary conditions:** Empty logs, single dive, max dives
- **Data integrity:** Compare parsed values against Python output

### Integration Tests
- **Full workflow:** Download → Parse → Render on real device
- **Edge cases:** Disconnection during download, corrupt packets
- **Multiple dives:** Verify sector offset calculations work correctly

### Visual Tests
- **SVG rendering:** Compare graphs to Python matplotlib output
- **Scalability:** Test at different viewport sizes, ensure SVG scales properly
- **UI responsiveness:** Test on mobile browsers (Chrome Android)
- **Accessibility:** Ensure readable fonts, sufficient contrast, text elements readable by screen readers
- **Interactivity:** Test hover effects and tooltips (if implemented)

## 7. Rollout Plan

### Phase 1: Core MVP (Recommended for initial PR)
- Reuse BLE download code from branch
- Implement JS binary parser
- Implement SVG renderer (string-based approach for simplicity)
- Add simple dive list UI
- Test with real device

### Phase 2: Polish
- Enhance with DOM-based SVG for interactivity
- Add hover tooltips showing depth/time details
- Add loading animations
- Error handling and retry logic
- Export to raw JSON (backup option)
- Export current SVG as downloadable .svg file
- Documentation updates

### Phase 3: Export Formats (Future)
- UDDF export
- Subsurface export
- CSV export for spreadsheets

## 8. Open Questions

1. **Branch strategy:** Revive `divelog_download_parse` or start fresh from `main`?
2. **Storage:** Should parsed dives be cached in memory during session?
3. **Large datasets:** Device might have 50+ dives - pagination needed?
4. **Mobile support:** SVG rendering on small screens - make responsive with viewBox?
5. **Interactivity:** Start with simple string-based SVG or implement DOM-based for tooltips in MVP?

## 9. Summary

This design provides a pure JavaScript, zero-dependency solution for dive log visualization. By porting the Python parsing logic to the browser and rendering with inline SVG, users get immediate feedback without installing tools or saving files. The architecture maintains the project's philosophy of being a single, self-contained HTML file that works entirely offline after initial load.

**Key Benefits of SVG Approach:**
- Declarative markup is easier to debug than imperative Canvas commands
- Scalable graphics work perfectly on any screen size or resolution
- Can leverage existing CSS variables and styling
- Easy to add interactivity (hover effects, tooltips) in Phase 2
- Can export graphs as .svg files for use in reports

**Implementation Size:**
- Reuse ~200 lines of BLE download code from existing branch
- Add ~300 lines of binary parsing logic
- Add ~100 lines of SVG rendering code (string-based)
- Add ~50 lines of UI orchestration

**Total: ~450 lines of new code** - keeping the addition modest and maintainable while delivering full dive log visualization capability.
