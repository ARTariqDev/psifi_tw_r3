# 🌍 Pakistan Earthquake Dashboard - Complete Guide

**Project Goal:** Build a real-time dashboard displaying earthquakes in Pakistan (magnitude ≥6.0) over the last 10 years using USGS API, NodeRED, MongoDB Atlas, and Grafana.

---

## 📋 Table of Contents
1. [Prerequisites](#prerequisites)
2. [MongoDB Atlas Setup](#mongodb-atlas-setup)
3. [NodeRED Setup](#nodered-setup)
4. [Grafana Setup](#grafana-setup)
5. [Testing Everything](#testing-everything)
6. [Presentation Tips](#presentation-tips)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### What You Need:
- ✅ NodeRED (already installed)
- ✅ Grafana (already installed at `/Users/tariqmahmood/Projects/grafana-12.3.1`)
- ⚠️ MongoDB Atlas account (we'll create this)
- ⚠️ Internet connection

### Pakistan Geographic Boundaries:
- **Latitude:** 23.5° N to 37.5° N
- **Longitude:** 60.5° E to 77.5° E

### USGS API Endpoint:
```
https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&starttime=2015-01-01&endtime=2025-12-28&minlatitude=23.5&maxlatitude=37.5&minlongitude=60.5&maxlongitude=77.5&minmagnitude=6.0
```

---

## MongoDB Atlas Setup

### Step 1: Create Account
1. Go to https://www.mongodb.com/cloud/atlas/register
2. Sign up with email or Google account
3. Complete verification

### Step 2: Create Free Cluster
1. Click **Build a Database**
2. Select **M0 Free Shared** cluster
3. Choose provider: **AWS**
4. Region: **Mumbai (ap-south-1)** or **Singapore (ap-southeast-1)**
5. Cluster Name: `pakistan-earthquakes`
6. Click **Create Cluster** (wait 3-5 minutes)

### Step 3: Database Access (Security)
1. Go to **Security** → **Database Access**
2. Click **Add New Database User**
3. Authentication Method: **Password**
   - Username: `earthquake_user`
   - Password: Click **Autogenerate Secure Password** (SAVE THIS!)
   - Or create your own: `EarthquakePK2025!`
4. Database User Privileges: **Read and write to any database**
5. Click **Add User**

### Step 4: Network Access (Allow Connections)
1. Go to **Security** → **Network Access**
2. Click **Add IP Address**
3. Click **Allow Access from Anywhere** (0.0.0.0/0)
4. Confirm

### Step 5: Get Connection String
1. Go to **Database** → Click **Connect** on your cluster
2. Choose **Connect your application**
3. Driver: **Node.js**
4. Copy the connection string:
```
mongodb+srv://earthquake_user:<password>@pakistan-earthquakes.xxxxx.mongodb.net/?retryWrites=true&w=majority
```
5. **Replace `<password>` with your actual password**
6. **SAVE THIS CONNECTION STRING** - you'll need it for NodeRED and Grafana!

### Step 6: Create Database & Collection
1. Click **Browse Collections**
2. Click **Add My Own Data**
3. Database name: `pakistan_earthquakes`
4. Collection name: `earthquakes`
5. Click **Create**

**✅ MongoDB Atlas Setup Complete!**

---

## NodeRED Setup

### Step 1: Start NodeRED
```bash
node-red
```
- Access at: http://localhost:1880
- Keep this terminal running!

### Step 2: Install MongoDB Node
1. Click menu (☰) → **Manage palette**
2. Go to **Install** tab
3. Search: `node-red-contrib-mongodb4`
4. Click **Install**
5. Confirm installation

### Step 3: Create the Flow

#### Visual Flow Structure:
```
[Inject] → [HTTP Request] → [Function] → [Split] → [MongoDB Out]
    ↓           ↓              ↓           ↓            ↓
  (Trigger) (Fetch API)  (Transform)  (Split)    (Store DB)
```

#### Node 1: Inject Node (Timer)
1. Drag **inject** node from palette
2. Double-click to configure:
   - **Payload:** timestamp
   - **Repeat:** interval (every 1 hour) - OR leave as manual for initial load
   - **Name:** "Trigger Earthquake Fetch"
3. Click **Done**

#### Node 2: HTTP Request Node (Fetch API)
1. Drag **http request** node from palette
2. Double-click to configure:
   - **Method:** GET
   - **URL:**
   ```
   https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&starttime=2015-01-01&endtime=2025-12-28&minlatitude=23.5&maxlatitude=37.5&minlongitude=60.5&maxlongitude=77.5&minmagnitude=6.0
   ```
   - **Return:** a parsed JSON object
   - **Name:** "Fetch Pakistan Earthquakes"
3. Click **Done**

#### Node 3: Function Node (Transform Data)
1. Drag **function** node from palette
2. Double-click to configure:
   - **Name:** "Transform Earthquake Data"
   - **Code:**
   ```javascript
   // Extract features array (individual earthquakes)
   const earthquakes = msg.payload.features;
   const transformed = [];

   for (let eq of earthquakes) {
       transformed.push({
           id: eq.id,
           magnitude: eq.properties.mag,
           location: eq.properties.place,
           time: new Date(eq.properties.time),
           depth_km: eq.geometry.coordinates[2],
           latitude: eq.geometry.coordinates[1],
           longitude: eq.geometry.coordinates[0],
           tsunami: eq.properties.tsunami,
           significance: eq.properties.sig,
           alert: eq.properties.alert || "none",
           url: eq.properties.url,
           felt_reports: eq.properties.felt || 0
       });
   }

   // Return transformed array
   return { payload: transformed };
   ```
3. Click **Done**

#### Node 4: Split Node (Split Array)
1. Drag **split** node from palette
2. No configuration needed - just add it
3. This splits the array so MongoDB stores individual documents

#### Node 5: MongoDB Out Node (Store to Atlas)
1. Drag **mongodb4 out** node from palette
2. Double-click to configure:
   - **Server:** Click pencil icon to add new server
     - **Name:** Atlas Cluster
     - **Protocol:** mongodb+srv://
     - **Hostname:** (from your connection string - e.g., `pakistan-earthquakes.xxxxx.mongodb.net`)
     - **Port:** leave empty for mongodb+srv
     - **Database:** `pakistan_earthquakes`
     - **Username:** `earthquake_user`
     - **Password:** (your password)
     - Click **Add**
   - **Operation:** insert
   - **Collection:** `earthquakes`
   - **Name:** "Store in Atlas"
3. Click **Done**

**Alternative - Using Full Connection String:**
If the above doesn't work, use the URI connection type:
- Click pencil icon on Server
- Choose **URI** tab
- Paste your full connection string:
  ```
  mongodb+srv://earthquake_user:YOUR_PASSWORD@pakistan-earthquakes.xxxxx.mongodb.net/pakistan_earthquakes?retryWrites=true&w=majority
  ```

#### Node 6: Debug Node (Optional - See Output)
1. Drag **debug** node from palette
2. Connect it to Function node output
3. This lets you see the transformed data

### Step 4: Connect the Nodes
1. Click and drag from the output dot of **Inject** → to input of **HTTP Request**
2. **HTTP Request** output → **Function** input
3. **Function** output → **Split** input
4. **Split** output → **MongoDB Out** input
5. (Optional) **Function** output → **Debug** input

### Step 5: Deploy the Flow
1. Click **Deploy** button (top right)
2. Should show "Successfully deployed"

### Step 6: Test the Flow
1. Click the button on the left of the **Inject** node
2. Watch the debug panel on the right
3. Should see data being processed
4. Check MongoDB Atlas:
   - Go to Atlas → Browse Collections
   - Select `pakistan_earthquakes` → `earthquakes`
   - You should see earthquake documents!

**✅ NodeRED Setup Complete!**

---

## Grafana Setup

### Step 1: Start Grafana
```bash
cd /Users/tariqmahmood/Projects/grafana-12.3.1
./bin/grafana server
```
- Access at: http://localhost:3000
- Default login: `admin` / `admin` (change password on first login)
- Keep this terminal running!

### Step 2: Install MongoDB Plugin
1. Go to **Configuration** (⚙️ icon) → **Plugins**
2. Search for "MongoDB"
3. Click on **MongoDB** data source
4. Click **Install**
5. Wait for installation to complete

### Step 3: Add MongoDB Atlas Data Source
1. Go to **Configuration** → **Data Sources**
2. Click **Add data source**
3. Search and select **MongoDB**
4. Configure:
   - **Name:** Pakistan Earthquakes DB
   - **MongoDB URL:** Your Atlas connection string:
     ```
     mongodb+srv://earthquake_user:YOUR_PASSWORD@pakistan-earthquakes.xxxxx.mongodb.net/
     ```
   - **Database:** `pakistan_earthquakes`
   - **Default Collection:** `earthquakes`
5. Click **Save & Test**
6. Should see green "Data source is working" message

### Step 4: Create Dashboard

#### Create New Dashboard:
1. Click **+** icon → **Dashboard**
2. Click **Add visualization**
3. Select **Pakistan Earthquakes DB** as data source

---

### Panel 1: Total Earthquake Count (Big Number)

**Configuration:**
1. **Query:**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$count": "total"
    }
  ]
}
```
2. **Visualization:** Stat
3. **Title:** "Total Earthquakes (Magnitude 6.0+)"
4. **Panel options:**
   - Text size: Auto
   - Graph mode: None
   - Color mode: Background
5. **Apply**

---

### Panel 2: Earthquakes Timeline (Line Graph)

**Configuration:**
1. **Query:**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$project": {
        "time": 1,
        "magnitude": 1
      }
    },
    {
      "$sort": { "time": 1 }
    }
  ]
}
```
2. **Visualization:** Time series
3. **Title:** "Earthquake Timeline (2015-2025)"
4. **Field config:**
   - X-axis: time
   - Y-axis: magnitude
5. **Apply**

---

### Panel 3: Magnitude Distribution (Bar Chart)

**Configuration:**
1. **Query:**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$bucket": {
        "groupBy": "$magnitude",
        "boundaries": [6.0, 6.5, 7.0, 7.5, 8.0],
        "default": "8.0+",
        "output": {
          "count": { "$sum": 1 }
        }
      }
    }
  ]
}
```
2. **Visualization:** Bar chart
3. **Title:** "Earthquakes by Magnitude Range"
4. **Apply**

---

### Panel 4: Pakistan Earthquake Map (Geomap)

**Configuration:**
1. **Query:**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$project": {
        "latitude": 1,
        "longitude": 1,
        "magnitude": 1,
        "location": 1,
        "depth_km": 1
      }
    }
  ]
}
```
2. **Visualization:** Geomap
3. **Title:** "At-Risk Areas - Earthquake Locations"
4. **Map view:**
   - View: Coordinates
   - Initial view: Set to Pakistan (lat: 30, lon: 70, zoom: 5)
5. **Layer:**
   - Type: Markers
   - Location: Coords
   - Latitude field: latitude
   - Longitude field: longitude
   - Size: Based on magnitude
   - Color: Based on magnitude (red = higher)
6. **Apply**

---

### Panel 5: Recent Major Earthquakes (Table)

**Configuration:**
1. **Query:**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$sort": { "time": -1 }
    },
    {
      "$limit": 15
    },
    {
      "$project": {
        "_id": 0,
        "Date": "$time",
        "Magnitude": "$magnitude",
        "Location": "$location",
        "Depth (km)": "$depth_km",
        "Alert": "$alert"
      }
    }
  ]
}
```
2. **Visualization:** Table
3. **Title:** "15 Most Recent Major Earthquakes"
4. **Apply**

---

### Panel 6: Statistics Panel (Stats Grid)

**Configuration:**
1. **Query (Average Magnitude):**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$group": {
        "_id": null,
        "avgMagnitude": { "$avg": "$magnitude" },
        "maxMagnitude": { "$max": "$magnitude" },
        "minMagnitude": { "$min": "$magnitude" },
        "avgDepth": { "$avg": "$depth_km" }
      }
    }
  ]
}
```
2. **Visualization:** Stat
3. **Title:** "Key Statistics"
4. Create multiple queries for each stat:
   - Average Magnitude
   - Maximum Magnitude
   - Average Depth
   - Earthquakes with Tsunami warnings
5. **Apply**

---

### Panel 7: Earthquakes per Year (Bar Chart)

**Configuration:**
1. **Query:**
```javascript
{
  "collection": "earthquakes",
  "aggregate": [
    {
      "$group": {
        "_id": { "$year": "$time" },
        "count": { "$sum": 1 }
      }
    },
    {
      "$sort": { "_id": 1 }
    },
    {
      "$project": {
        "year": "$_id",
        "count": 1,
        "_id": 0
      }
    }
  ]
}
```
2. **Visualization:** Bar chart (horizontal)
3. **Title:** "Earthquakes per Year"
4. **Apply**

---

### Step 5: Arrange Dashboard
1. Drag panels to arrange in logical layout
2. Resize panels as needed
3. Suggested layout:
```
┌─────────────────┬─────────────────┬─────────────────┐
│  Total Count    │  Max Magnitude  │  Avg Depth      │
├─────────────────┴─────────────────┴─────────────────┤
│           Earthquake Timeline (full width)          │
├─────────────────────────────┬─────────────────────┤
│   Pakistan Earthquake Map   │  Magnitude          │
│        (Geomap)             │  Distribution       │
├─────────────────────────────┴─────────────────────┤
│        Recent Major Earthquakes (Table)             │
├─────────────────────────────┬─────────────────────┤
│   Earthquakes per Year      │   Key Statistics    │
└─────────────────────────────┴─────────────────────┘
```

### Step 6: Configure Dashboard Settings
1. Click ⚙️ (settings) icon at top
2. **General:**
   - Name: "Pakistan Earthquake Dashboard - Magnitude 6.0+"
   - Description: "Real-time monitoring of major earthquakes in Pakistan (2015-2025)"
   - Tags: earthquakes, pakistan, usgs
3. **Time options:**
   - Timezone: Asia/Karachi
   - Auto refresh: 5m (or 1m for demo)
4. **Save dashboard**

**✅ Grafana Dashboard Complete!**

---

## Testing Everything

### Complete System Test:

#### Terminal Setup:
```bash
# Terminal 1 - NodeRED
node-red

# Terminal 2 - Grafana
cd /Users/tariqmahmood/Projects/grafana-12.3.1
./bin/grafana server
```

#### Step-by-Step Test:

1. **Test MongoDB Atlas Connection:**
   - Go to MongoDB Atlas website
   - Browse Collections → `pakistan_earthquakes` → `earthquakes`
   - Should see earthquake documents (if you ran NodeRED flow)

2. **Test NodeRED Flow:**
   - Open http://localhost:1880
   - Click inject node button
   - Watch debug panel
   - Check MongoDB Atlas - should see new/updated data

3. **Test Grafana Connection:**
   - Open http://localhost:3000
   - Go to Configuration → Data Sources → Pakistan Earthquakes DB
   - Click "Save & Test"
   - Should show green success message

4. **View Dashboard:**
   - Open your dashboard
   - All panels should show data
   - Try different time ranges
   - Test auto-refresh

### Expected Results:
- **~15-30 earthquakes** in Pakistan over last 10 years (magnitude 6.0+)
- **Highest concentration:** Northern Pakistan, Kashmir region, Hindu Kush
- **Notable event:** 2015 Hindu Kush earthquake (7.5 magnitude)
- **At-risk areas:** Along Himalayan fault lines, northern regions

---

## Presentation Tips

### For Tomorrow's Competition:

#### Before Presentation:
1. ✅ Run both NodeRED and Grafana
2. ✅ Verify internet connection (needed for MongoDB Atlas)
3. ✅ Test dashboard - make sure all panels load
4. ✅ Have backup: Screenshot your dashboard
5. ✅ Prepare explanation (see below)

#### What to Explain (2-3 minutes):
1. **Topic Choice:**
   - "I chose earthquake monitoring for Pakistan because it's in a high-risk seismic zone"
   - "Focused on magnitude 6.0+ earthquakes over last 10 years"

2. **Data Flow:**
   ```
   USGS API → NodeRED → MongoDB Atlas → Grafana Dashboard
   ```
   - "NodeRED fetches data from USGS earthquake API"
   - "Data is stored in MongoDB Atlas cloud database"
   - "Grafana visualizes the data in real-time"

3. **Dashboard Features:**
   - Geographic map showing at-risk areas
   - Timeline showing earthquake frequency
   - Statistics: magnitude distribution, depth analysis
   - Table of recent major earthquakes

4. **Why MongoDB Atlas:**
   - "Cloud database provides better scalability than Google Sheets"
   - "Professional solution used in production systems"
   - "Handles real-time queries efficiently"
   - "Added complexity for better marks"

5. **Insights from Data:**
   - "Northern Pakistan is most at-risk"
   - "X earthquakes magnitude 6.0+ in last 10 years"
   - "Average depth: Y km"
   - "Most active years: [mention specific years]"

#### Backup Plan:
- Record a screen recording of your working dashboard
- If internet fails, show the video
- Have screenshots ready

---

## Troubleshooting

### NodeRED Issues:

**MongoDB node not found:**
```bash
cd ~/.node-red
npm install node-red-contrib-mongodb4
# Restart NodeRED
```

**Connection timeout to Atlas:**
- Check your internet connection
- Verify IP whitelist in Atlas (0.0.0.0/0)
- Check password in connection string (no special URL encoding needed)

**HTTP Request fails:**
- Test API in browser first
- Check date ranges are valid
- Verify geographic coordinates

### Grafana Issues:

**MongoDB plugin not showing:**
- Restart Grafana after installing plugin
- Check if plugin installed: Configuration → Plugins

**Data source connection fails:**
- Verify connection string format
- Test connection in MongoDB Compass first
- Check Atlas network access settings

**No data in panels:**
- Verify collection name is correct
- Check MongoDB has data: use Atlas Browse Collections
- Try simpler query first: `{}`

**Geomap not showing points:**
- Verify latitude/longitude fields exist in data
- Check field names match in query
- Ensure coordinates are numbers, not strings

### MongoDB Atlas Issues:

**Can't connect from anywhere:**
- Make sure 0.0.0.0/0 is in Network Access whitelist
- Wait a few minutes after adding IP

**Authentication failed:**
- Double-check username: `earthquake_user`
- Verify password (no extra spaces)
- Make sure user has "Read and write" privileges

**Cluster paused:**
- Atlas pauses free clusters after 60 days inactivity
- Click "Resume" button

---

## Quick Reference Commands

### Start Everything:
```bash
# Terminal 1
node-red

# Terminal 2
cd /Users/tariqmahmood/Projects/grafana-12.3.1
./bin/grafana server
```

### Access URLs:
- **NodeRED:** http://localhost:1880
- **Grafana:** http://localhost:3000 (admin/admin)
- **MongoDB Atlas:** https://cloud.mongodb.com

### API Endpoint:
```
https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&starttime=2015-01-01&endtime=2025-12-28&minlatitude=23.5&maxlatitude=37.5&minlongitude=60.5&maxlongitude=77.5&minmagnitude=6.0
```

---

## Next Steps

1. ✅ Complete MongoDB Atlas setup
2. ✅ Create NodeRED flow
3. ✅ Run flow to populate data
4. ✅ Set up Grafana data source
5. ✅ Build dashboard panels
6. ✅ Test everything
7. ✅ Practice presentation
8. 🎯 **Present tomorrow!**

---

## Additional Resources

- **USGS API Docs:** https://earthquake.usgs.gov/fdsnws/event/1/
- **NodeRED Docs:** https://nodered.org/docs/
- **MongoDB Atlas Docs:** https://docs.atlas.mongodb.com/
- **Grafana Docs:** https://grafana.com/docs/

---

**Good luck with your presentation! 🚀**
