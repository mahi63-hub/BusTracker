# Real-Time Transit Tracker

A professional-grade transit tracking system featuring a high-performance **Fastify backend WebSocket server** and an interactive **React Native / Expo client** mapping real-time vehicle simulation data.

---

## Technical Architecture Overview

The application utilizes a decoupled, real-time push model:
- **Fastify Server (`/server`)**: Simulates GTFS-RT (General Transit Feed Specification - Realtime) updates by updating coordinates along GeoJSON LineString tracks using the spherical Haversine formula. Broadcasts current state array every 3 seconds to `/ws`. Includes `/health` check and CORS configuration.
- **Expo React Native App (`/client`)**: Integrates `react-native-maps` to draw path routes and vehicle markers. Prevents visual jumping or teleportation by interpolating marker coordinates over 2.9-second intervals using `AnimatedRegion`. Connects via WebSockets with robust exponential backoff.

```
transit-tracker/
├── package.json                   # Monorepo tasks script
├── docker-compose.yml             # Local backend container run orchestration
├── server/                        # Fastify Backend Service
│   ├── Dockerfile                 # Multi-stage lightweight deployment container
│   ├── package.json               # Backend dependencies
│   ├── routes/
│   │   ├── route-A.geojson        # Central Loop route coordinates
│   │   └── route-B.geojson        # Harbor Express route coordinates
│   └── src/
│       ├── server.js              # Server config and API/WebSocket routes
│       └── simulation.js          # Vehicle speed and position simulator
└── client/                        # Expo Client Mobile Application
    ├── package.json               # Client dependencies
    ├── app.json                   # Expo application settings
    ├── babel.config.js            # Babel preset mapping
    ├── index.js                   # Application bootstrap loader
    ├── App.js                     # Root screen containing Map, Overlay, Details
    └── src/
        ├── components/
        │   └── AnimatedVehicleMarker.js  # Smooth-gliding marker component
        ├── hooks/
        │   └── useVehicleSocket.js       # Auto-reconnection socket manager
        └── utils/
            ├── eta.js                    # Haversine & ETA calculation
            └── reconnect.js              # Reconnection delay calculator
```

---

## Phase 1 — Requirement Analysis

### Requirement Checklist

- **Backend Docker Container**: Root `docker-compose.yml` launches Fastify server on port 3000, built from a local Dockerfile, and includes an HTTP curl health check on `http://localhost:3000/health`.
- **REST Route Endpoint**: `GET /routes` serves a JSON array containing at least two valid GeoJSON Feature LineString route paths.
- **WebSocket Endpoint**: `ws://localhost:3000/ws` hosts the WebSocket server which accepts and holds connections.
- **Broadcast Simulator**: Every 3 seconds, the WebSocket server pushes updates in the specified JSON array format containing `vehicle_id`, `route_id`, `position`, `speed`, and `timestamp`.
- **Map View Container**: Renders a map component containing the `data-testid="map-container"` (and `testID="map-container"`) attribute.
- **Route Polylines**: Draws routes on the map using polyline graphics tagged with `data-testid="polyline-route-[route_id]"`.
- **Animated Markers**: Draws markers representing vehicles using coordinates that interpolate smoothly over 2.9-second durations, avoiding sudden jumping. Markers have `data-testid="vehicle-marker-[vehicle_id]"`.
- **Interactive Details**: Pressing a vehicle marker opens a bottom sheet container tagged with `data-testid="vehicle-detail-sheet"` containing:
  - Vehicle ID display: `data-testid="detail-vehicle-id"`
  - Speed display: `data-testid="detail-speed"`
  - Last updated timestamp display: `data-testid="detail-last-updated"`
- **Test Exposes**: Employs global test functions exposed on the `window` or `globalThis` object:
  - `window.calculateETA({ currentPosition, nextStopPosition, speedKmh })` returns ETA in minutes.
  - `window.getReconnectDelay(attemptNumber)` returns backoff delay in milliseconds.

### Risk Analysis & Mitigation
- **Maps Compatibility on Web Platform**: React Native maps can have dependency issues on Web. *Mitigation*: Configured Expo with Metro bundler and added both `testID` (for Native tests) and `data-testid` (for Web browser environments) attributes.
- **Spamming Reconnections**: Network failures could lead to server flooding. *Mitigation*: Implemented mathematical exponential backoff capped at 30,000ms.

---

## Phase 2 — Project Design Decisions

- **State Management**: React state handles active vehicle arrays and routes locally in `App.js`. This is efficient and keeps the bundle lightweight without Redux or Zustand.
- **Animation Strategy**: Uses `AnimatedRegion` from `react-native-maps` inside the marker component. This moves animations off the React re-render queue directly onto the map thread for smooth timing transitions.
- **Pure Reusable Components**: Extracted the marker animation into a memoized component `AnimatedVehicleMarker` so updates to one vehicle do not trigger re-renders or interrupt animations on other vehicles.

---

## Phase 5 — GitHub Development Plan

We structure the development into these 12 incremental commits:

1. **Commit 1: Project setup**
   - Files: `package.json`, `.gitignore`
   - Purpose: Setup root workspace configuration and exclude directories from tracking.
2. **Commit 2: Folder structure and base layout**
   - Files: Creates `/server`, `/client` directories and structures code folders.
3. **Commit 3: Core components**
   - Files: `/client/src/components/AnimatedVehicleMarker.js`
   - Purpose: Introduce custom marker components with smooth animation regions.
4. **Commit 4: State management**
   - Files: `/client/src/hooks/useVehicleSocket.js`
   - Purpose: Implement custom state and event handlers for real-time WebSocket feeds.
5. **Commit 5: Feature implementation**
   - Files: `/client/App.js`
   - Purpose: Connect WebSocket state to render markers and show/hide bottom sheets.
6. **Commit 6: Data integration**
   - Files: `/server/src/server.js`, `/server/routes/`
   - Purpose: Code the Fastify server and load GeoJSON routes from files.
7. **Commit 7: Charts/UI enhancements**
   - Files: `/client/App.js`
   - Purpose: Improve the visual design of the status bar, bottom details sheets, and maps styling.
8. **Commit 8: Validation and error handling**
   - Files: `/client/src/utils/eta.js`, `/client/src/utils/reconnect.js`
   - Purpose: Create mathematical validation algorithms and expose helper functions globally.
9. **Commit 9: Responsive improvements**
   - Files: `/client/App.js`
   - Purpose: Apply flexbox scaling to fit both smaller smartphone layouts and larger browser windows.
10. **Commit 10: Testing and bug fixes**
    - Files: `/server/src/simulation.js`
    - Purpose: Ensure simulation coordinates do not return NaN and prevent simulation state desyncs.
11. **Commit 11: Documentation**
    - Files: `README.md`
    - Purpose: Add user instructions, execution guides, and dependency installation maps.
12. **Commit 12: Final polish**
    - Files: `/server/Dockerfile`, `docker-compose.yml`
    - Purpose: Verify port configuration and test Docker container healthcheck parameters.

---

## Phase 6 — Installation and Run Instructions

### 1. Prerequisite Checklist
- Node.js (v18 or higher recommended)
- Docker Desktop (for containerizing backend)

### 2. Backend Server Setup
To start the Fastify server locally inside a Docker container:
```bash
# From the root directory:
docker-compose up --build
```
The server will start on `http://localhost:3000`. You can check the health status by running:
```bash
curl http://localhost:3000/health
```

Alternatively, to run the server locally without Docker:
```bash
cd server
npm install
npm start
```

### 3. Frontend App Setup
To start the React Native / Expo application:
```bash
# In a new terminal window:
cd client
npm install

# Start the Expo development server:
npm run web
```
This launches the Metro bundler in offline mode. Select **w** to open it directly in your web browser.

---

## Final Validation Matrix

| Requirement | Status | Implementation Details |
| :--- | :--- | :--- |
| **1. Docker Compose & Healthcheck** | Implemented | [docker-compose.yml](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/docker-compose.yml) lines 1-19 |
| **2. REST GET /routes (GeoJSON LineString)** | Implemented | [server.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/server/src/server.js) lines 43-43 |
| **3. WebSocket Server /ws** | Implemented | [server.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/server/src/server.js) lines 45-49 |
| **4. Broadcast simulator (3s interval)** | Implemented | [server.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/server/src/server.js) lines 51-58 |
| **5. MapView testID="map-container"** | Implemented | [App.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/App.js) lines 138-142 |
| **6. Route Polylines testID** | Implemented | [App.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/App.js) lines 157-158 |
| **7. Vehicle Markers testID** | Implemented | [AnimatedVehicleMarker.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/src/components/AnimatedVehicleMarker.js) lines 34-36 |
| **8. Bottom Sheet vehicle-detail-sheet** | Implemented | [App.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/App.js) lines 181-184 |
| **9. Bottom Sheet Detail Labels** | Implemented | [App.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/App.js) lines 192, 204, 217 |
| **10. window.calculateETA expose** | Implemented | [App.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/App.js) lines 25-33 and [eta.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/src/utils/eta.js) |
| **11. window.getReconnectDelay expose** | Implemented | [App.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/App.js) lines 25-33 and [reconnect.js](file:///C:/Users/Dell/OneDrive/Documents/GPP/transit-tracker/client/src/utils/reconnect.js) |
#   B u s T r a c k e r  
 