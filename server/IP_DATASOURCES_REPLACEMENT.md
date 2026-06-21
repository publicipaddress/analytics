# Public IP Resolution & Geolocation Architecture Report

This document maps all database files, loaders, utility functions, and consumer services in the backend (`server/`) responsible for extracting, mapping, and resolving public IP address information (including ASN, geolocation, and bot classification).

---

### 1. Core Database Files & Packages
These files store the binary MaxMind GeoIP/GeoLite databases and provide the dependency to parse them:
*   **ASN Database File:** `/workspace/analytics/server/GeoLite2-ASN.mmdb` (Resolves IP to Autonomous System Numbers).
*   **City/Location Database File:** `/workspace/analytics/server/GeoLite2-City.mmdb` (Resolves IP to City, Country, Region, Lat/Lon, and Timezone).
*   **MaxMind Parser Dependency:** Defined in `/workspace/analytics/server/package.json` as `"@maxmind/geoip2-node": "6.1.0"`.
*   **Docker Container Layer:** In `/workspace/analytics/server/Dockerfile` (Lines 44-45), these databases are copied into the final production container.

---

### 2. Database Loaders & Types (The Data-Access Layer)
The primary loaders and abstraction wrappers are under `/workspace/analytics/server/src/db/geolocation/`:

| File Path | Function/Export | Purpose |
| :--- | :--- | :--- |
| `types.ts` | `LocationResponse` | Standard type definition for city, country, region, countryIso, lat/lon, and timeZone. |
| `geolocation.ts` | `getLocation(ips: string[])` | Loads `GeoLite2-City.mmdb`, processes an array of IP addresses, and returns a key-value record mapping each IP address to its resolved geographical attributes. |
| `asn.ts` | `lookupAsn(ip: string)` | Loads `GeoLite2-ASN.mmdb`, parses an IP address, and returns Autonomous System Number (ASN) and Organization data. |

---

### 3. IP Address Identification Utility
Before any lookup can happen, the backend must identify the authentic public IP address of the client:
*   **File:** `/workspace/analytics/server/src/utils.ts`
*   **Function:** `getIpAddress(request: FastifyRequest): string`
*   **Logic:** Uses `X-Real-IP`, `CF-Connecting-IP`, `X-Forwarded-For`, and finally `request.ip`.

---

### 4. Consumer Services & Event Pipelines (The Processing Layer)

#### 📊 Pageviews & Analytical Event Tracking
*   **`server/src/services/tracker/pageviewQueue.ts`**: Uses `getLocation(ips)` to append geography attributes to Clickhouse event records.
*   **`server/src/services/tracker/trackEvent.ts`**: Uses `getLocation([requestIP])` to verify if a request's `countryIso` is excluded from site tracking.

#### 🤖 Bot Detection & Blocking
*   **`server/src/services/tracker/botBlocking/index.ts`**: Uses `lookupAsn(ipForAsn)` in bot heuristics to classify hosting/cloud infrastructure IPs.
*   **`server/src/services/tracker/botBlocking/botEventQueue.ts`**: Uses `getLocation(ips)` to resolve source geolocation for incident logging.
*   **`server/src/services/tracker/botBlocking/index.test.ts`**: Unit-test mocks for `lookupAsn`.

#### 🎥 Session Replays
*   **`server/src/api/sessionReplay/recordSessionReplay.ts`**: Uses `getLocation([requestIP])` to restrict recording based on country configuration.
*   **`server/src/services/replay/trackingUtils.ts`**: Uses `getLocation([ipAddress])` to enrich metadata during replay parsing.

#### 🎏 Feature Flag Rules
*   **`server/src/api/featureFlags/index.ts`**: Uses `getLocation([ipAddress])` for location-based feature flag targeting.
