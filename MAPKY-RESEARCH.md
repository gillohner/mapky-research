# MapKy: Decentralized Social Layer on OpenStreetMap

> **Status**: WIP Research & design document (Feb 2026). No implementation yet — this captures the data model architecture, competitive analysis, and design decisions for future reference.

## Context

MapKy aims to be a Pubky-based decentralized alternative to Google Maps/Apple Maps by adding a **social layer on top of OpenStreetMap**. OSM provides excellent geographic data (83%+ road coverage globally, millions of POIs) but explicitly stores only objective "ground truth" — it has **no reviews, ratings, social features, or subjective place data**. This is the single biggest gap preventing OSM-based apps (Organic Maps, OsmAnd) from competing with proprietary maps. The Open Reviews Association (Mangrove Reviews) is nascent and centralized. MapKy fills this gap using Pubky's decentralized identity and storage.

---

## OSM Foundation: How References Work

OSM has 3 primitives: **Node** (point, lat/lon), **Way** (ordered list of nodes), **Relation** (groups of elements). Each has a type + numeric ID (64-bit int). IDs are **not unique across types** — `node/123` and `way/123` are different objects.

**Canonical reference format**: `{type}/{id}` — e.g. `node/1573053883`, `way/987654321`, `relation/111111`
**URL format**: `https://www.openstreetmap.org/node/1573053883`
**API format**: `https://api.openstreetmap.org/api/0.6/node/1573053883`

OSM IDs are **stable** (survive edits) but elements can be deleted. The existing `Location` struct in pubky-app-specs already extracts `(osm_type, osm_id)` from OSM URLs via `location.rs:118-129`. (Eventky extension)

### OSM Tagging System

OSM uses free-form `key=value` tags on any element. No schema enforcement — community conventions documented on OSM Wiki. 99,000+ distinct keys, 168M+ distinct tags.

**Common categories**: `amenity=restaurant`, `building=residential`, `highway=motorway`, `shop=supermarket`, `tourism=hotel`, `natural=water`

**Places/POIs typically include**: name, addr:housenumber, addr:street, phone, website, opening_hours, cuisine, wheelchair

### OSM Data Access

- **Overpass API**: Read-only queries by location/type/tags (overpass-turbo.eu)
- **Nominatim**: Geocoding and reverse geocoding (text → coords, coords → address)
- **Tile servers**: Raster (pre-rendered PNG) and vector (client-rendered, more flexible)
- **Main API**: Direct element access `api/0.6/{type}/{id}`

---

### Key Competitors & Projects

**OSM-based apps (no social layer)**:
- **Organic Maps** — Fast, privacy-focused, offline. No reviews. Community-funded, open-source.
- **OsmAnd** — Feature-rich, offline, plugin system. No reviews. Slant #1 for Android offline.
- **Magic Earth** — EU-based, privacy-focused. Commercial.

**Street-level imagery**:
- **Mapillary** — Acquired by Meta (2020). CC BY-SA 4.0 images. Community fears shutdown.
- **KartaView** — Telenav→Grab. 55TiB backup of Mapillary data. Free, CC BY-SA.
- **Hivemapper** — Blockchain/crypto incentivized. Dashcam 4K imagery. HONEY tokens.

**Social/review attempts**:
- **Open Reviews Association / Mangrove Reviews** — Nascent open-source review standard. MapComplete integration. Goal: OsmAnd/Organic Maps plugins.
- **OSM Notes** — Built-in annotation for map data quality only, not social reviews.

**Activity/route apps**:
- **AllTrails** — 450K+ trails, AI route creation, heat mapping. Subscription model.
- **Komoot** — Turn-by-turn navigation, multi-sport. Strong sharing features.

**Check-in apps**:
- **Foursquare/Swarm** — Check-ins, badges, mayorships, streaks, lifetime stats. Declining but still unique.

---

## Data Model Architecture

Namespace: `mapky.app/` (following `pubky.app/`, `eventky.app/` pattern)

### OsmRef (Embedded Struct, not standalone)

Canonical reference to an OSM element. Embedded in reviews, tags, photos, etc. — not stored at its own path. Follows the pattern of `PubkyAppPostEmbed` (which stores just a URI, not cached content).

```rust
pub enum OsmElementType { Node, Way, Relation }

pub struct OsmRef {
    pub osm_type: OsmElementType,  // node, way, or relation
    pub osm_id: i64,               // 64-bit OSM numeric ID
}
```

- `canonical()` returns `"node/123456789"`
- `osm_url()` returns `"https://www.openstreetmap.org/node/123456789"`
- Validates: `osm_id > 0`

### Models (MVP)

#### MapkyAppPost — `/pub/mapky.app/posts/<id>` (TimestampId)

Unified type for all user content about a place — reviews, questions, comments, replies. Mirrors `PubkyAppPost` (which has `PubkyAppPostKind`) but anchored to an OSM element. One flexible type covers everything, including Q&A (no separate Question/Answer models needed).

```rust
pub enum MapkyAppPostKind {
    Review,    // has rating — "4 stars, great pasta". Indexer aggregates ratings.
    Question,  // asking about a place — "Is there parking nearby?". Indexer builds Q&A threads.
    Comment,   // general comment or reply. Used with `parent` for answers to questions, replies to reviews.
}

pub struct MapkyAppPost {
    pub place: OsmRef,                    // OSM element this post is about
    pub kind: MapkyAppPostKind,           // tells the indexer and UI how to handle this post
    pub content: Option<String>,          // text content (review text, question, comment, caption)
    pub rating: Option<u8>,              // 1-10 (half-stars). Only meaningful for kind=Review.
    pub attachments: Option<Vec<String>>, // pubky:// file URIs (reuses PubkyAppFile — photos, videos, PDFs, audio)
    pub parent: Option<String>,          // pubky:// URI of parent MapkyAppPost (for replies, answers)
}
```

What this single type covers:
- **Review**: `kind=Review` + `rating` + `content` — "4.5 stars, great pasta"
- **Review with photos**: `kind=Review` + `rating` + `content` + `attachments`
- **Question**: `kind=Question` + `content` — "Is there parking nearby?"
- **Answer**: `kind=Comment` + `parent` pointing to a Question post — "Yes, there's a lot behind the building"
- **Comment on a review**: `kind=Comment` + `parent` pointing to a Review post
- **Photo/media post**: `kind=Comment` + `attachments` with optional `content` as caption
- **Any nesting depth** via `parent` chains

**Tagging — posts and individual files**: `PubkyAppTag` targets any `pubky://` URI, so both posts and their individual attachments are taggable:
- Tag a review: `PubkyAppTag{uri: "pubky://user/.../posts/ID", label: "helpful"}` — "helpful", "outdated", "spam"
- Tag an attached file: `PubkyAppTag{uri: "pubky://user/.../files/FILE_ID", label: "menu"}` — "menu", "food", "interior", "exterior"

This enables per-file gallery categorization without a separate media model. The indexer builds the place gallery by collecting all file URIs from posts about that place, and resolves tags on each file for UI filtering/categorization. A review with 3 photos = 1 post write + 3 PubkyAppFile writes; other users can then tag each photo independently.

#### MapkyAppLocationTag — `/pub/mapky.app/location_tags/<id>` (HashId)

Subjective crowdsourced labels on places. Each user can apply each label to each place once. **The count of distinct taggers IS the signal strength** — no upvote/downvote needed.

Two design options under consideration:

**Option A: Flat labels (like PubkyAppTag)**

```rust
pub struct MapkyAppLocationTag {
    pub place: OsmRef,
    pub label: String,              // same rules as PubkyAppTag (max 20 chars, lowercase)
    pub created_at: i64,
}
```

- Pros: Simple, fully open-ended, community self-organizes
- Cons: Fragmented labels ("wheelchair-ok" vs "wheelchair-friendly" vs "accessible"), hard to build structured UIs, no easy way to aggregate by theme, harder to build features like safety heat maps
- HashId = `Blake3("{osm_canonical}:{label}")`

**Option B: Categorized tags**

```rust
pub enum LocationTagCategory {
    Safety,        // dangerous, well-lit, safe, sketchy
    Accessibility, // wheelchair-friendly, step-free, blind-friendly
    Ambiance,      // romantic, noisy, quiet, cozy, lively
    Family,        // family-friendly, kid-safe, stroller-friendly
    Pets,          // dog-friendly, pet-free
    Food,          // vegan, halal, kosher, gluten-free
    Sustainability,// eco-friendly, zero-waste, organic
    Service,       // fast-service, slow-service, friendly-staff
    Value,         // overpriced, good-value, budget-friendly
}

pub struct MapkyAppLocationTag {
    pub place: OsmRef,
    pub category: LocationTagCategory,  // required category
    pub label: String,                  // free-text within category (max 20 chars, lowercase)
    pub created_at: i64,
}
```

- Pros: Enables structured UX — safety heat map overlay, accessibility filter, "show all ambiance tags for this place". Categories give the indexer clear dimensions to aggregate on. Predefined categories guide users toward useful labels instead of noise.
- Cons: Category enum must evolve over time (new categories = spec update). Users may disagree on which category a tag belongs to. Slightly more complex.
- HashId = `Blake3("{osm_canonical}:{category}:{label}")` — one tag per user per place per category+label combo

**Option B extension: Category ratings**

Categories could optionally carry a per-category star rating, giving structured sub-ratings alongside the overall review score:

```rust
pub struct MapkyAppLocationTag {
    pub place: OsmRef,
    pub category: LocationTagCategory,  // required category
    pub label: String,                  // free-text within category (max 20 chars, lowercase)
    pub rating: Option<u8>,            // optional 1-10 per-category rating
    pub created_at: i64,
}
```

This enables: "Safety: 8/10 + 'well-lit'" or "Service: 3/10 + 'slow-service'". The indexer can aggregate average ratings per category per place — giving a breakdown like Google Maps' sub-ratings (food, service, atmosphere) but crowdsourced and richer.

HashId would be `Blake3("{osm_canonical}:{category}:{label}")` — the rating is mutable (user can change their rating by overwriting the same hash path) while the tag identity stays stable.

**Recommendation**: Option B (with optional category ratings). The category constraint unlocks high-value features:
- Safety heat map: aggregate all `Safety` tags + ratings by geo region, overlay on map for travelers
- Accessibility dashboard: filter places by `Accessibility` tags
- Food dietary filters: show only `Food:vegan` tagged places
- Per-category tag clouds on place cards
- Sub-rating breakdown: "Safety 4.2, Service 3.8, Value 4.5" per place

**UX integration with MapkyAppPost reviews**: When a user writes a review (MapkyAppPost with rating), the UI can automatically suggest relevant tag categories based on the OSM element type:
- `amenity=restaurant` → prompt for Food, Service, Value, Ambiance categories
- `tourism=hotel` → prompt for Service, Value, Ambiance, Accessibility
- `leisure=park` → prompt for Safety, Family, Pets, Accessibility
- `shop=*` → prompt for Service, Value
- `highway=*` / paths → prompt for Safety, Accessibility

This makes the review flow feel like a guided experience — "Rate your visit, then tell us about specific aspects" — while the data model stays clean (review is a MapkyAppPost, category tags are separate MapkyAppLocationTags created alongside it). The UI orchestrates; the spec doesn't couple them.
- The free-text label within each category still allows open-ended expression

#### MapkyAppCollection — `/pub/mapky.app/collections/<id>` (TimestampId)

Named lists of places. Competes with Google Maps Lists and Apple Maps Guides.

```rust
pub struct MapkyAppCollection {
    pub name: String,                         // 1-100 chars
    pub description: Option<String>,          // max 2000 chars
    pub items: Vec<OsmRef>,                   // ordered, max 500
    pub image_uri: Option<String>,            // cover image
}
```

- Items stored inline (Vec) — single atomic write, index = display order
- All collections are public (homeserver storage is public). Private collections can be revisited when homeservers support private storage.

#### MapkyAppIncident — `/pub/mapky.app/incidents/<id>` (TimestampId)

Waze-style crowdsourced incident/hazard reports. Geo-located, time-sensitive. Key difference from MapkyAppPost: incidents are coordinate-anchored (not OSM element-anchored) and have an expiry concept.

```rust
pub enum IncidentType {
    Accident,       // vehicle collision
    Hazard,         // road hazard, fallen tree, debris
    RoadClosure,    // construction, event closure
    Police,         // speed trap, checkpoint
    Flooding,       // water on road
    IceSnow,        // winter conditions
    PoorVisibility, // fog, smoke
    Danger,         // general safety concern (not road-specific)
    Other,
}

pub enum IncidentSeverity {
    Low,       // minor inconvenience
    Medium,    // significant delay or caution needed
    High,      // major hazard, avoid area
}

pub struct MapkyAppIncident {
    pub incident_type: IncidentType,
    pub severity: IncidentSeverity,
    pub lat: f64,                         // incident location (required)
    pub lon: f64,                         // incident location (required)
    pub heading: Option<f64>,             // direction of travel when reporting (0-360)
    pub description: Option<String>,      // max 500 chars
    pub attachments: Option<Vec<String>>, // pubky:// file URIs (photos of the incident)
    pub expires_at: Option<i64>,          // suggested expiry (Unix microseconds). Indexer may auto-expire.
}
```

- Coordinate-anchored, not OSM-element-anchored (an accident is at a point on the road, not "about" a restaurant)
- `expires_at` is a suggestion — the indexer decides actual TTL based on incident type (accidents expire faster than road closures)
- Indexer aggregates: multiple users reporting the same incident at similar coordinates = higher confidence
- Users can "confirm" an existing incident by creating their own at nearby coordinates (indexer clusters these)
- Naturally decays: old incidents without fresh confirmations get lower confidence scores

#### MapkyAppGeoCapture — `/pub/mapky.app/geo_captures/<id>` (TimestampId)

Any geo-located media contribution anchored to coordinates in space — not tied to a specific OSM element. This is about building the map itself, not commenting on places. Decentralized alternative to Mapillary/KartaView/Google Street View.

The actual media lives in `PubkyAppFile` + `PubkyAppBlob` which already handle any IANA MIME type. The GeoCapture is the spatial metadata wrapper. Supported media includes:

- **Photos**: street-level, aerial, facade (`image/*`)
- **360° panoramas**: equirectangular or cubemap (`image/*` with viewer hint)
- **Video**: walkthroughs, dashcam, drone footage (`video/*`)
- **3D models**: photogrammetry, scans (`model/gltf-binary`, `model/gltf+json`, `model/obj`, `model/stl`)
- **Point clouds**: LiDAR captures (`application/octet-stream` or domain-specific)
- **Audio**: ambient recordings, soundscapes (`audio/*`)

```rust
/// Hint for how the capture should be rendered/viewed
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum GeoCaptureKind {
    Photo,          // standard photo
    Panorama,       // 360° / spherical — viewer should use panorama renderer
    Video,          // standard video
    Video360,       // 360° video — viewer should use spherical video renderer
    Model3d,        // 3D model — viewer should use 3D renderer (glTF, OBJ, etc.)
    PointCloud,     // LiDAR / photogrammetry point cloud
    Audio,          // ambient audio recording
    Other,          // anything else — fall back to content_type from PubkyAppFile
}

pub struct MapkyAppGeoCapture {
    pub file_uri: String,             // pubky:// URI to PubkyAppFile (content_type determines format)
    pub kind: GeoCaptureKind,         // rendering hint (complements content_type for viewer selection)
    pub lat: f64,                     // capture latitude (required)
    pub lon: f64,                     // capture longitude (required)
    pub ele: Option<f64>,             // elevation in meters (useful for drone/aerial captures)
    pub heading: Option<f64>,         // compass bearing 0-360 (0=North)
    pub pitch: Option<f64>,           // camera pitch -90 to 90 (0=horizontal)
    pub fov: Option<f64>,             // field of view in degrees (for photos/video, helps rendering)
    pub caption: Option<String>,      // max 500 chars
    pub sequence_uri: Option<String>, // pubky:// URI of MapkyAppRoute or another grouping for "next/prev"
    pub sequence_index: Option<u32>,  // position within sequence (0-indexed)
}
```

**Key distinction from MapkyAppPost**: Posts are *about* OSM elements (reviews, comments). GeoCaptures are raw spatial media *at coordinates* — they contribute to the map's visual/spatial layer rather than the social layer. A GeoCapture might later get linked to an OSM element by the indexer (nearest POI matching), but the capture itself is coordinate-anchored.

**GeoCaptureKind vs content_type**: `PubkyAppFile.content_type` tells you the file format (`image/jpeg`, `model/gltf-binary`). `GeoCaptureKind` tells the UI *how to render it* — a JPEG could be a regular photo or a 360° panorama; a video could be flat or spherical. The kind is the rendering hint.

#### MapkyAppRoute — `/pub/mapky.app/routes/<id>` (TimestampId)

User-created routes (hiking, cycling, etc.). Competes with AllTrails/Komoot.

```rust
pub enum RouteActivityType { Hiking, Cycling, Running, Walking, Driving, Skiing, Other }
pub enum RouteDifficulty { Easy, Moderate, Difficult, Expert }
pub struct Waypoint { pub lat: f64, pub lon: f64, pub ele: Option<f64> }

pub struct MapkyAppRoute {
    pub name: String,                        // max 200 chars
    pub description: Option<String>,         // max 10000 chars
    pub activity: RouteActivityType,
    pub difficulty: Option<RouteDifficulty>,
    pub waypoints: Vec<Waypoint>,            // max 10000
    pub osm_ways: Option<Vec<OsmRef>>,       // optional semantic OSM links
    pub distance_m: Option<f64>,
    pub elevation_gain_m: Option<f64>,
    pub elevation_loss_m: Option<f64>,
    pub estimated_duration_s: Option<i64>,
    pub image_uri: Option<String>,
}
```

---

## Path & ID Summary

| Model | Path | ID Strategy | Rationale |
|---|---|---|---|
| MapkyAppPost | `/pub/mapky.app/posts/<id>` | TimestampId | Unified reviews/comments/media, threaded via parent |
| MapkyAppLocationTag | `/pub/mapky.app/location_tags/<id>` | HashId | Idempotent per user+place+label (Option A) or +category+label (Option B) |
| MapkyAppCollection | `/pub/mapky.app/collections/<id>` | TimestampId | Editable |
| MapkyAppIncident | `/pub/mapky.app/incidents/<id>` | TimestampId | Time-sensitive, coordinate-anchored |
| MapkyAppGeoCapture | `/pub/mapky.app/geo_captures/<id>` | TimestampId | Chronological |
| MapkyAppRoute | `/pub/mapky.app/routes/<id>` | TimestampId | Editable |

---

## Reuse of Existing pubky-app-specs Code

| What | Reuses |
|---|---|
| File/media storage | `PubkyAppFile` + `PubkyAppBlob` (unchanged) — attachments in MapkyAppPost reference these |
| Tag label validation | `sanitize_tag_label()`, `validate_tag_label()` from `src/models/tag.rs` |
| Post threading & Q&A | `MapkyAppPost.parent` mirrors `PubkyAppPost.parent` — answers are just comments on question posts |
| Post kind | `MapkyAppPostKind` mirrors `PubkyAppPostKind` pattern for differentiating content types |
| OSM URL parsing | Extends `Location.osm_id()` pattern from `src/models/location.rs` |
| Trait implementations | `Validatable`, `TimestampId`/`HashId`, `HasIdPath` from `src/traits.rs` |
| Tagging posts + files | `PubkyAppTag` targets any URI — tag posts ("helpful", "spam") and individual attached files ("menu", "interior") for gallery categorization |

### PubkyAppTag: Supported Taggable Models in UI

`PubkyAppTag` targets any `pubky://` URI, so the UI should support tagging all five primary content models. Each tag is a `PubkyAppTag{uri, label, created_at}` written to the tagger's homeserver.

| Model | Example URI | Example Labels | UI Context |
|---|---|---|---|
| **MapkyAppPost** | `pubky://user/.../posts/ID` | "helpful", "outdated", "spam", "detailed", "funny" | Tag reviews, questions, comments — enables community moderation and quality signals |
| **PubkyAppFile** | `pubky://user/.../files/FILE_ID` | "menu", "food", "interior", "exterior", "parking" | Tag individual photos/media attached to posts — enables per-file gallery categorization and filtering |
| **MapkyAppCollection** | `pubky://user/.../collections/ID` | "date-night", "food-spots", "hidden-gems", "kid-friendly", "budget" | Tag curated place lists — enables discovery of collections by theme |
| **MapkyAppGeoCapture** | `pubky://user/.../geo_captures/ID` | "panorama", "street-art", "damage-report", "seasonal", "construction" | Tag geo-located media — enables spatial media filtering and map layer categorization |
| **MapkyAppRoute** | `pubky://user/.../routes/ID` | "scenic", "wheelchair-accessible", "family-friendly", "technical", "flat" | Tag routes — enables route discovery by characteristic beyond activity type and difficulty |

**Indexer implications**: The indexer should resolve `PubkyAppTag` URIs against all five models and maintain tag counts per content item. This enables tag-based filtering in search results, feeds, and map overlays (e.g. "show only geo captures tagged 'street-art'", "find collections tagged 'date-night'", "routes tagged 'scenic'").

---

## MapKy Indexer

The mapky-indexer watches Pubky homeserver events for `mapky.app/` paths and maintains a geo-optimized data store focused on place-centric aggregation and spatial queries.

### Data store options (TBD)

- **PostgreSQL + PostGIS** — Mature, excellent spatial indexing, SQL familiarity
- **SQLite + SpatiaLite** — Simpler, single-file, good for prototyping
- **Redis with geo commands** — Fast for simple nearby queries, limited for complex spatial ops
- **Tile38** — Dedicated geospatial database, real-time geofencing
- Could use a combination: PostGIS for primary storage + Redis for hot aggregation caches

### Watcher pattern

Subscribes to homeserver PUT/DEL events for `mapky.app/` paths:
- `PUT mapky.app/posts/*` → parse post, dispatch by `kind`: Review → aggregate ratings; Question → index as Q&A thread; Comment → link to parent thread; Media → index in place gallery
- `PUT mapky.app/location_tags/*` → increment tag count for place (by category if Option B)
- `DEL mapky.app/location_tags/*` → decrement tag count
- etc.

### Aggregation

- Per-place: avg_rating, review_count, tag counts (label->count by category), file_count, geo_capture_count
- Per-user: their reviews, collections, check-in history
- Spatial: places within bounding box, nearby with radius, clustering at zoom levels

### API endpoints (TBD)

- `GET /v0/place/{osm_type}/{osm_id}` — aggregated place data
- `GET /v0/place/{osm_type}/{osm_id}/reviews` — paginated reviews
- `GET /v0/place/{osm_type}/{osm_id}/tags` — tag counts
- `GET /v0/place/{osm_type}/{osm_id}/photos` — paginated photos
- `GET /v0/place/{osm_type}/{osm_id}/questions` — Q&A threads
- `GET /v0/user/{id}/reviews` — user's reviews
- `GET /v0/user/{id}/collections` — user's collections
- `GET /v0/nearby?lat=X&lon=Y&radius=Z` — geo queries
- `GET /v0/bbox?n=X&s=X&e=X&w=X` — bounding box queries (for map viewport)
- `GET /v0/search?q=...&lat=X&lon=X` — place search with location bias

---

## Known UX Gaps vs Google Maps / Apple Maps

Features that users expect from a maps app but are **not covered by the data models above**. Categorized by where they'd need to be addressed.

### Data model gaps (future spec additions)
- **Business info** (hours, phone, menu, prices) — OSM has some (`opening_hours`, `phone`, `website`) but it's incomplete. Could add a MapkyAppBusinessClaim model for owners to enrich their listing, or rely on OSM + community edits. Ideally if something like this would be implemented the added data would be stored in OSM itself (via the indexer writing back to OSM API) rather than in a separate Pubky model, to keep the social layer cleanly separated from the objective map data. This makes us storing it on Pubky homeservers more of a caching layer for display rather than the source of truth which is kind of redundant so probably not worth it.
- **Saved/favorite places** — partially covered by MapkyAppCollection, but a lightweight "star" or bookmark (single place, no list) might warrant its own model or could just be a single-item collection that UI treats as a favorite. The indexer could maintain a "favorites" list per user by aggregating these.

### App/platform concerns (not spec)
- **Transit routing** (#1 gap) — requires GTFS feeds from transit agencies. App-level integration, not a data model. Libraries: OpenTripPlanner, Navitia, Transitous.
- **Real-time traffic** — requires telemetry aggregation or external feeds. App-level. Could use incident reports (MapkyAppIncident) as a partial signal. Future anonymous or opt-in location telemetry could enhance this. Uber/Bolt like services could be built on top of this with driver apps updating their locations. There are a lot of Privacy concerns here, so this would need to be designed carefully with user consent and data minimization in mind. If possible with anonymized, aggregated data rather than individual location tracking.
- **Search quality** — ranking, autocomplete, fuzzy matching. Indexer + Nominatim tuning. Photon (Komoot's geocoder) is a strong OSM-native option.
- **Offline maps** — tile caching, offline routing. App-level concern. MapLibre supports offline tile packages. MBTiles format.
- **Turn-by-turn navigation** — OSRM or Valhalla for routing engine. App-level integration.
- **Indoor maps** — OSM has Simple Indoor Tagging but coverage is minimal. Long-term.

### External data dependencies (third-party)
- **Satellite/aerial imagery** — tile providers: Mapbox, Thunderforest, Stamen, or self-hosted. Not a data model concern.
- **Weather overlays** — external API (OpenWeatherMap, etc.)
- **Popular times / busyness** — would require opt-in location telemetry or check-in aggregation. Privacy-sensitive; needs careful design if ever pursued.

---

## App Approach: Open Questions

### Option A: Fork an Existing OSM App

**Organic Maps** (C++, cross-platform):
- Pros: Mature rendering, offline maps, fast, already has favorites/bookmarks
- Cons: C++ codebase, no web version, adding Pubky SDK (Rust/JS) to C++ is complex

**OsmAnd** (Java/Kotlin, Android):
- Pros: Plugin system, most features, Android-first
- Cons: Android-only natively, heavy codebase, Java interop with Pubky Rust SDK

### Option B: Web App with PWA

- Pros: Cross-platform from day one, easy Pubky SDK integration (WASM), modern JS tooling, faster iteration, can use MapLibre GL JS or Leaflet for rendering
- Cons: Map rendering performance vs native, offline support more limited, no app store presence initially

### Option C: React Native with `@synonymdev/react-native-pubky`

A React Native wrapper around `pubky-core` already exists: [`@synonymdev/react-native-pubky`](https://github.com/pubky/react-native-pubky/)

- Pros: Native performance, cross-platform mobile, existing Pubky SDK integration, can use `@maplibre/maplibre-react-native` for maps
- Cons: More complex build setup, web support via React Native Web is possible but secondary, smaller ecosystem than pure web, desktop support would require additional effort (Electron or Tauri wrapper)

### Option D: Tauri Desktop + Capacitor Mobile

- Pros: Rust backend (natural Pubky fit), web frontend, desktop + mobile
- Cons: Smaller ecosystem, mobile Tauri is still maturing

---

## References

- [OSM Elements Wiki](https://wiki.openstreetmap.org/wiki/Elements)
- [OSM Tags Wiki](https://wiki.openstreetmap.org/wiki/Tags)
- [OSM Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API)
- [OSM Nominatim](https://wiki.openstreetmap.org/wiki/Nominatim)
- [Open Reviews Association](https://wiki.openstreetmap.org/wiki/Open_Reviews_Association)
- [MapComplete](https://wiki.openstreetmap.org/wiki/MapComplete)
- [Mapillary](https://en.wikipedia.org/wiki/Mapillary)
- [KartaView](https://wiki.openstreetmap.org/wiki/KartaView)
- [Hivemapper](https://beemaps.com/blog/how-a-new-global-decentralized-map-will-revolutionize-the-mapping-economy/)
- [Foursquare Swarm](https://swarmapp.com/)
- [AllTrails vs Komoot](https://www.theplanetedit.com/alltrails-vs-komoot/)
- [Apple Maps Expert Reviews](https://www.apple.com/newsroom/2025/05/apple-brings-insights-ratings-and-reviews-from-expert-sources-to-apple-maps/)
- [Google Popular Times](https://blog.google/products-and-platforms/products/maps/maps101-popular-times-and-live-busyness-information/)
- [Google Street View Contributions](https://blog.google/products/maps/anyone-can-share-their-world-with-street-view/)
- [OSM Data Completeness](https://pmc.ncbi.nlm.nih.gov/articles/PMC5552279/)
- [react-native-pubky](https://github.com/pubky/react-native-pubky/) — React Native wrapper for pubky-core via UniFFI (Synonym)
