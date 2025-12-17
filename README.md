# **Deepblu Cosmiq 5 \- Web Controller**

## **🌊 Project Overview**

This project provides a web-based controller for the **Deepblu Cosmiq 5** dive computer. Since the official app has been discontinued, this tool allows you to modify all settings on your device directly from a web browser using Bluetooth Low Energy (BLE).

It runs entirely in your browser (client-side)—no data is sent to any external server.

🤖 Vibe Coding Disclaimer  
This entire project was "vibecoded" with Gemini Pro 3\. The code was generated through an iterative conversation between a human operator (providing logs and testing) and the AI (analyzing patterns and writing code). It represents a collaborative effort to preserve legacy hardware through modern AI assistance.

## **🚀 How to Use**

### **1\. Requirements**

* **Hardware:** Deepblu Cosmiq 5 Dive Computer.  
* **Browser:** You need a browser that supports Web Bluetooth.  
  * **PC/Mac/Android:** Google Chrome, Edge, or Opera.  
  * **iOS (iPhone/iPad):** Safari **does not** support this. You must download a dedicated WebBLE browser like **Bluefy** from the App Store.

### **2\. Connection Steps**

1. Open the [Hosted Web Page](https://blue-notes-robot.github.io/cosmiq5-web/) (or your local index.html).  
2. Turn on your Cosmiq 5 and ensure Bluetooth is active.  
3. Click the big blue **"Connect & Sync"** button.  
4. Select your device (usually named COSMIQ or Deepblu) from the list.  
5. Wait for the green "Connected" status. The app will automatically read your current settings.

### **3\. Features**

* **General:** Set Time/Date format, Sync Time, Units (Metric/Imperial), Backlight intensity, Screen Timeout, and Power-Saving (Eco) mode.
* **Environment:** Configure High Altitude mode or High Salinity mode (Advanced).
* **Scuba:** Configure Air Mix (Nitrox), PPO2, Depth Alarms, Time Alarms, and Safety Factor.
* **Freedive:** Configure Max Time and 6 distinct Depth Alarms.
* **Dive History:** View and download your dive logs with detailed profiles:
  * Fast header-only scanning to see dive list (date, depth, duration)
  * On-demand individual dive profile downloads (prevents browser crashes)
  * Interactive depth/time charts for each dive
  * Persistent storage in IndexedDB (survives browser refresh)
  * Sort by date (most recent first)
  * Navigate through all dives with Previous/Next buttons
* **Diagnostics:** A "Byte Hunter" tab allows you to see the raw data packets coming from the device.

### **4\. Using Dive History**

1. Navigate to the **Dive History** tab after connecting to your device.
2. Click **"Scan Dive Headers"** to quickly load the list of all dives (shows date, max depth, duration).
3. Browse through your dives using the **Previous/Next** buttons.
4. For any dive, click **"Download Dive Profile"** to fetch the detailed depth/time data.
5. View the interactive chart showing your depth profile over time.
6. Use **"↻ Reload Dive Profile"** if you need to re-fetch the data.

**Note:** Dive data is stored locally in your browser (IndexedDB), so it persists across sessions. Your dive history is automatically sorted by date with the most recent dive first.

## **🛠️ Technical Details**

### **Dive History Implementation**

The dive history feature uses a two-phase approach:
- **Phase 1:** Fast header-only scan downloads 36-byte headers for all dives (showing metadata only)
- **Phase 2:** On-demand body data download for individual dives (fetches detailed depth samples)

This prevents browser crashes when handling large dive logs by only downloading full profile data when needed.

### **Data Storage**

- Uses IndexedDB for persistent local storage
- Dive headers and samples stored separately per dive
- No data sent to external servers (100% client-side)

### **Protocol Details**

- BLE communication using Web Bluetooth API
- 36-byte dive headers (not 72 as initially assumed)
- 4-byte depth samples: [Marker_Low][Marker_High][Depth_Low][Depth_High]
- Depth values in centimeters (divide by 100 for meters)

## **⚠️ Important Safety Warning**

**This software is UNOFFICIAL and experimental.**

Diving involves significant risks, including decompression sickness, oxygen toxicity, and drowning.

1. **ALWAYS** verify your settings on the physical device screen before entering the water.
2. **NEVER** rely solely on this software to configure life-safety parameters.
3. The authors and the AI assistant accept **NO RESPONSIBILITY** for malfunctions, incorrect settings, or any safety incidents resulting from the use of this tool.

*Dive safe. Always carry a backup.*

## **📝 Recent Updates**

### Dive History Features (December 2024)
- ✅ Fixed header parsing to correctly read all dives (36-byte headers, not 72)
- ✅ Implemented on-demand individual dive profile downloads
- ✅ Added date-based sorting (most recent first)
- ✅ Fixed mode parsing (Scuba vs Freedive vs Gauge)
- ✅ Added persistent storage with IndexedDB
- ✅ Interactive depth/time charts with SVG rendering
- ✅ Reload functionality for dive profiles
