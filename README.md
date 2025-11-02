# wmeGisLBBOX Script Documentation

A reference guide for the internal APIs, key methods, and utility functions in the `wmeGisLBBOX` userscript.

---

## Table of Contents

- [wmeGisLBBOX Script Documentation](#wmegislbbox-script-documentation)
  - [Table of Contents](#table-of-contents)
  - [Class: wmeGisLBBOX](#class-wmegislbbox)
    - [Constructor](#constructor)
      - [`new wmeGisLBBOX()`](#new-wmegislbbox)
  - [File and Folder Convention Summary:](#file-and-folder-convention-summary)
  - [Methods](#methods)
    - [fetchJsonWithCache(url)](#fetchjsonwithcacheurl)
    - [getCountriesAndSubsJson()](#getcountriesandsubsjson)
      - [Top-level Country Data (`COUNTRIES_BBOX_ESPG4326.json`)](#top-level-country-data-countries_bbox_espg4326json)
      - [USA First-Level Subdivisions (`USA_BBOX_ESPG4326.json`)](#usa-first-level-subdivisions-usa_bbox_espg4326json)
      - [Country Second-Level Subdivisions (Example: USA -\> Alaska)](#country-second-level-subdivisions-example-usa---alaska)
    - [getIntersectingStatesAndCounties(viewportBbox, highPrecision = false, returnGeoJson = false)](#getintersectingstatesandcountiesviewportbbox-highprecision--false-returngeojson--false)
    - [getIntersectingSubdivisions(countryObj, viewportBbox)](#getintersectingsubdivisionscountryobj-viewportbbox)
    - [cleanIntersectingData(intersectingCountries)](#cleanintersectingdataintersectingcountries)
    - [fetchAndCheckGeoJsonIntersection(countyCode, subCode, subSubCode, viewportBbox, returnGeoJson = false)](#fetchandcheckgeojsonintersectioncountycode-subcode-subsubcode-viewportbbox-returngeojson--false)
    - [Master method: `whatsInView(viewportBbox, highPrecision = false, returnGeoJson = false)`](#master-method-whatsinviewviewportbbox-highprecision--false-returngeojson--false)
  - [Utility Functions (Internal)](#utility-functions-internal)
    - [checkIntersection(bbox1, bbox2)](#checkintersectionbbox1-bbox2)
    - [isPointInPolygon(point, vs)](#ispointinpolygonpoint-vs)
    - [segmentIntersection(p1, p2, q1, q2)](#segmentintersectionp1-p2-q1-q2)
    - [hasIntersection(polygon1, polygon2)](#hasintersectionpolygon1-polygon2)
  - [Examples](#examples)
    - [Find What Regions are in View](#find-what-regions-are-in-view)
    - [Fetch Counties Intersecting with Viewport](#fetch-counties-intersecting-with-viewport)
  - [Error Handling](#error-handling)
  - [License](#license)
  - [Additional Notes](#additional-notes)
- [WME whatsInView (Script Plug-in) Overview](#wme-whatsinview-script-plug-in-overview)

---

## Class: wmeGisLBBOX

### Constructor

#### `new wmeGisLBBOX()`

Instantiates a new wmeGisLBBOX object. Designed to encapsulate the caching layer and provide instance methods for geographic intersection logic.

```javascript
const bboxApi = new wmeGisLBBOX();
```

---

## File and Folder Convention Summary:

- **Top-level countries:**  
  `/BBOX JSON/COUNTRIES_BBOX_ESPG4326.json`  
  _Country keys (e.g., `'USA'`, `'CAN'`, `'FRA'`) in this file are used for naming subfolders and as prefixes for country-specific files._

  > **Note:**  
  > The `Sub_level` property indicates the maximum administrative subdivision depth available for that country:
  >
  > - `Sub_level: 0` means only the country (no subdivisions)
  > - `Sub_level: 1` means country + first-level subdivisions (such as states, provinces, territories, regions)
  > - `Sub_level: 2` includes country, first-level, and second-level (such as counties, districts, departments)
  > - `Sub_level: 3` includes a further third-level (such as cities, municipalities), with level 3 subdivisions _embedded within_ the second-level JSON files if present

- **First-level subdivisions (sub-level-1):**  
  `/BBOX JSON/<ISO_ALPHA3>/<ISO_ALPHA3>_BBOX_ESPG4326.json`  
  _Within each country folder, this file lists first-level subdivisions, keyed by codes like 'CA', 'TX' for USA. These keys are used for subsequent filenames._

- **Second-level subdivisions (sub-level-2):**  
  `/BBOX JSON/<ISO_ALPHA3>/<ISO_ALPHA3>-<SubL1 Key>_BBOX_ESPG4326.json`  
  _A file for each first-level subdivision, named with the country code and the sub-level-1 key (e.g., `USA-CA_BBOX_ESPG4326.json` for California). This file contains an array of objects, each keyed by a second-level subdivision name (e.g., county, borough, department)._

- **Third-level subdivisions (sub-level-3, if present):**  
  _Third-level subdivision data is included as a `subdivisions` array inside the relevant second-level objects in the sub-level-2 JSON. These deeper subdivisions do **not** have separate files._

**How this convention works:**

1. The primary country file (`COUNTRIES_BBOX_ESPG4326.json`) yields supported countries and their subdivision depth (via `Sub_level`), which establishes folder and file naming conventions as well as which levels are available.
2. Each country's sub-level-1 file (`<ISO_ALPHA3>_BBOX_ESPG4326.json`) lists first-level subdivisions (e.g., states for USA, provinces for Canada), using their keys (like `"CA"`, `"TX"`, `"ON"`) for naming second-level JSON files.
3. Each sub-level-2 file (`<ISO_ALPHA3>-<SubL1 Key>_BBOX_ESPG4326.json`) details second-level subdivisions. If there are third-level subdivisions, these appear as an array within the corresponding second-level object.
4. The convention applies identically for other countries; substitute ISO codes and subdivision keys as appropriate.

**Examples:**

- **United States (USA):**
  - First level:  
    `/BBOX JSON/USA/USA_BBOX_ESPG4326.json`
  - Second level (e.g., Alaska):  
    `/BBOX JSON/USA/USA-AK_BBOX_ESPG4326.json`
    - _Third-level (e.g., cities within Alaska boroughs) appear as `subdivisions` arrays inside each borough object in `USA-AK_BBOX_ESPG4326.json`._
- **Canada (CAN):**
  - First level:  
    `/BBOX JSON/CAN/CAN_BBOX_ESPG4326.json`
  - Second level (Ontario):  
    `/BBOX JSON/CAN/CAN-ON_BBOX_ESPG4326.json`
    - _Third-level (if present) is embedded inside second-level objects._
- **France (FRA):**
  - First level:  
    `/BBOX JSON/FRA/FRA_BBOX_ESPG4326.json`
  - Second level (Île-de-France):  
    `/BBOX JSON/FRA/FRA-IDF_BBOX_ESPG4326.json`

---

This hierarchical and predictable structure enables you to locate bounding box data for any administrative unit by following the keys from each parent subdivision level, and also lets you know which administrative depths are available based on the `Sub_level` property for each country.

---

---

## Methods

---

### fetchJsonWithCache(url)

**Purpose:**  
Fetches JSON data from a URL, applying a cache to reduce duplicate remote requests.

**Signature:**  
`fetchJsonWithCache(url: string): Promise<Object>`

**Usage Example:**

```js
bboxApi.fetchJsonWithCache('https://<username>.github.io/<repo>/data/countries.json').then((data) => {
  console.log(data);
});
```

---

### getCountriesAndSubsJson()

**Purpose:**  
Fetches and augments country data with subdivision information, returning a structure where each country is mapped to its first-level subdivisions, which may link further to deeper levels.

**Signature:**  
`getCountriesAndSubsJson(): Promise<Object>`

**How it works:**

- Loads country metadata from:  
  `https://wazedev.github.io/wmeGisLBBOX/BBOX%20JSON/COUNTRIES_BBOX_ESPG4326.json`
- For each country, loads first-level subdivisions (e.g. US states) from:  
  `https://wazedev.github.io/wmeGisLBBOX/BBOX%20JSON/USA/USA_BBOX_ESPG4326.json`
  (substitute `USA` with `<ISO_ALPHA3>` for other countries)
- For each first-level subdivision (e.g. 'AK'), additional files further describe lower subdivisions:  
  Example for Alaska counties/districts:  
  `https://wazedev.github.io/wmeGisLBBOX/BBOX%20JSON/USA/USA-AK_BBOX_ESPG4326.json`

**Error Handling:**

- Logs warnings if files fail to load; fills missing levels with empty objects.

---

#### Top-level Country Data (`COUNTRIES_BBOX_ESPG4326.json`)

```json
{
  "<ISO_ALPHA3>": {
    "name": "<country name>",
    "id": "<ISO_ALPHA3>",
    "ISO_ALPHA2": "<ISO_ALPHA2>",
    "ISO_ALPHA3": "<ISO_ALPHA3>",
    "Sub_level": <max subdivision level (integer)>,
    "bbox": [
      { "minLon": <number>, "minLat": <number>, "maxLon": <number>, "maxLat": <number> }
    ]
  },
  ...
}
```

---

#### USA First-Level Subdivisions (`USA_BBOX_ESPG4326.json`)

Each key is a state or territory code (e.g. "AK" for Alaska):

```json
{
  "AK": {
    "name": "Alaska",
    "sub_id": "AK",
    "sub_num": "02",
    "bbox": {
      "minLon": -179.231086,
      "minLat": 51.175092,
      "maxLon": 179.859681,
      "maxLat": 71.439786
    }
  },
  "CA": {
    "name": "California",
    "sub_id": "CA",
    "sub_num": "06",
    "bbox": {
      "minLon": -124.482003,
      "minLat": 32.529508,
      "maxLon": -114.131211,
      "maxLat": 42.009503
    }
  },
  "...": "..."
}
```

**Fields:**

- `name`: State or territory name
- `sub_id`: State code (same as key)
- `sub_num`: Numeric FIPS code
- `bbox`: Bounding box

---

#### Country Second-Level Subdivisions (Example: USA -> Alaska)

Each state has its own file, e.g.:`BBOX JSON/USA/USA-AK_BBOX_ESPG4326.json`

**Structure:**
An **array** of objects.  
Each object has a key for a second-level subdivision (borough/county/district), with its info and further subdivisions.

```json
[
  {
    "Ketchikan Gateway": {
      "sub_num": "130",
      "name": "Ketchikan Gateway",
      "bbox": {
        "minLon": -132.277239,
        "minLat": 54.648906,
        "maxLon": -129.974167,
        "maxLat": 56.406005
      },
      "subdivisions": [
        {
          "sub_num": "39010",
          "name": "Ketchikan",
          "bbox": {
            "minLon": -132.277239,
            "minLat": 54.648906,
            "maxLon": -129.974167,
            "maxLat": 56.406005
          }
        }
      ]
    }
  },
  {
    "Kenai Peninsula": {
      "sub_num": "122",
      "name": "Kenai Peninsula",
      "bbox": {
        "minLon": -154.748861,
        "minLat": 58.65274,
        "maxLon": -148.563026,
        "maxLat": 61.426239
      },
      "subdivisions": [
        {
          "sub_num": "38460",
          "name": "Kenai-Cook Inlet",
          "bbox": {
            "minLon": -154.748861,
            "minLat": 58.65274,
            "maxLon": -149.782754,
            "maxLat": 61.426239
          }
        },
        {
          "sub_num": "68610",
          "name": "Seward-Hope",
          "bbox": {
            "minLon": -151.002241,
            "minLat": 59.282299,
            "maxLon": -148.563026,
            "maxLat": 61.043124
          }
        }
      ]
    }
  }
  // ... more districts/counties
]
```

**Fields:**

- **2nd-level key** (e.g. _Ketchikan Gateway_, _Northwest Arctic_)
  - `sub_num`: Numeric FIPS code
  - `name`: Subdivision name
  - `bbox`: Bounding box
  - `subdivisions`: Array of deeper subdivisions (3rd-level, e.g. cities, if present)

---

**Other Countries:**  
Similar conventions. For Canada:

- Provinces in `CAN_BBOX_ESPG4326.json`
- Per-province subdivisions in `CAN-ON_BBOX_ESPG4326.json` (for Ontario)

---

**Practical Example:**

```js
// Load all countries and their subdivision meta-data
const countries = await bboxApi.getCountriesAndSubsJson();

// Get United States first-level subdivisions (states/territories)
const usaStates = countries['USA'].subL1; // Example: { AK: {...}, CA: {...}, ... }
const alaska = usaStates['AK'];

// To get Alaska's second-level divisions (boroughs/districts):
const akSubL2Arr = await bboxApi.fetchJsonWithCache('https://wazedev.github.io/wmeGisLBBOX/BBOX%20JSON/USA/USA-AK_BBOX_ESPG4326.json');

// Example: Find bounding box for Ketchikan Gateway in Alaska
// (akSubL2Arr is an array where each object uses the district's name as key)
const ketchikanObj = akSubL2Arr.find((obj) => obj['Ketchikan Gateway']);
const ketchikanGatewayInfo = ketchikanObj['Ketchikan Gateway'];
console.log('Ketchikan Gateway BBOX:', ketchikanGatewayInfo.bbox);

// If a third-level (city/area) exists, it's inside the .subdivisions array:
if (ketchikanGatewayInfo.subdivisions && ketchikanGatewayInfo.subdivisions.length > 0) {
  for (const sub3 of ketchikanGatewayInfo.subdivisions) {
    console.log(`Third-level: ${sub3.name}`, sub3.bbox);
  }
}
```

---

> **Notes:**
>
> - To traverse deeper, always follow the keys from the previous level's JSON.
> - The presence of a `subdivisions` array within a second-level object indicates available third-level data (no separate files for sub-level-3).
> - The availability of subdivision depths for a country can be checked via its `Sub_level` property in `COUNTRIES_BBOX_ESPG4326.json`.

---

### getIntersectingStatesAndCounties(viewportBbox, highPrecision = false, returnGeoJson = false)

**Purpose:**  
Identifies which U.S. states, counties (and sub-counties, if present) intersect with a specified viewport bounding box.  
Optionally applies high-precision GeoJSON shape intersection to improve accuracy for smaller or irregular regions.

**Signature:**  
`getIntersectingStatesAndCounties(viewportBbox: Object, highPrecision?: boolean, returnGeoJson?: boolean): Promise<Object>`

**Parameters:**

- `viewportBbox` (Object):  
  The bounding box to check intersection against, formatted as:  
  `{ minLon: number, minLat: number, maxLon: number, maxLat: number }`
- `highPrecision` (boolean, default=`false`):  
  If true, uses full GeoJSON shapes for intersection tests at the county (and below) level. This is slower and more accurate.
- `returnGeoJson` (boolean, default=`false`):  
  If true (with highPrecision enabled), attaches intersecting GeoJSON data features to results for use in mapping/visualization.

---

**Process Overview:**

1. Loads all U.S. state bounding boxes from `USA_BBOX_ESPG4326.json`.
2. Checks which states intersect the input viewport.
3. For each intersecting state:
   - Loads that state's county/district bounding boxes (e.g. `USA-TX_BBOX_ESPG4326.json`).
   - Checks which counties intersect the viewport.
   - For each intersecting county:
     - Checks for third-level (sub-county) subdivisions:
       - If present, filters sub-counties by intersection with the viewport.
       - (Third-level is only present in some counties and is embedded as an array.)
     - If `highPrecision` is true:
       - Fetches GeoJSON geometry and refines intersection check (replaces bbox filtering).
       - If `returnGeoJson` and the geometry truly intersects, attaches its GeoJSON feature.
     - Adds the intersecting county (and its sub-counties, if any) to the results, noting whether bbox or geojson was the source.
4. Returns a hierarchical object mapping state names to their intersecting counties and sub-counties.

---

**Returned Object Structure:**

```js
{
  "Texas": {
    subL1_id: "TX",
    subL1_num: "48",
    subL2: {
      "Harris": {
        subL2_num: "201",
        subL3: {
          "Houston": {
            subL3_num: "39010",
            source: "BBOX"
          }
          // ...other sub-counties if present
        },
        source: "BBOX", // or "GEOJSON"
        geoJsonData: {...} // Only present if returnGeoJson === true & highPrecision === true & intersects
      },
      // ...other counties
    },
    source: "BBOX" // or "GEOJSON" if any county used geojson
  },
  // ...other intersecting states
}
```

- If no sub-counties intersect, `subL3` is omitted or empty.

---

**Usage Example:**

```js
const viewport = { minLon: -102, minLat: 29, maxLon: -94, maxLat: 36 }; // Texas region
const results = await bboxApi.getIntersectingStatesAndCounties(viewport, true, false);
// results['Texas'].subL2 contains only counties that intersect the viewport
// Each county lists intersecting sub-counties (level-3) under .subL3
```

**With GeoJSON output:**

```js
const results = await bboxApi.getIntersectingStatesAndCounties(viewport, true, true);
console.log(results['Texas'].subL2['Harris'].geoJsonData); // GeoJSON feature if intersection confirmed
```

---

**Notes:**

- This method is only supported for the **USA**; other countries use [`getIntersectingSubdivisions`](#getintersectingsubdivisionscountryobj-viewportbbox).
- All fetches use caching to reduce network load.
- Handles anti-meridian wrapping and standard bbox formats.
- **Performance:** High precision mode fetches additional GeoJSON and is slower but more accurate; use for fine-scale mapping.
- **Error Handling:** If data is missing or fetch fails, the function logs an error and continues processing.

**See Also:**

- [getCountriesAndSubsJson()](#getcountriesandsubsjson)
- [getIntersectingSubdivisions()](#getintersectingsubdivisionscountryobj-viewportbbox)

---

### getIntersectingSubdivisions(countryObj, viewportBbox)

**Purpose:**  
For countries other than USA, identifies which first-level and (if available) second-level subdivisions intersect the specified viewport bounding box.

**Signature:**  
`getIntersectingSubdivisions(countryObj: Object, viewportBbox: Object): Promise<Object>`

**Parameters:**

- `countryObj` (Object):  
  An object describing the country, including its ISO code and subdivision depth. Must include at least:
  - `ISO_ALPHA3` (string): The 3-letter country code (e.g., "FRA" for France).
  - `Sub_level` (number): Maximum supported subdivision depth (see [File and Folder Convention Summary](#file-and-folder-convention-summary)).
- `viewportBbox` (Object):  
  Bounding box to check intersection against. Format:  
  `{ minLon: number, minLat: number, maxLon: number, maxLat: number }`

---

**Process Overview:**

1. Checks the country's `Sub_level` property to determine available subdivision depth.
2. If `Sub_level` >= 1:
   - Loads the country's **first-level subdivision data** from  
     `/BBOX JSON/<ISO_ALPHA3>/<ISO_ALPHA3>_BBOX_ESPG4326.json`
   - Iterates over all first-level subdivisions:
     - Checks if each subdivision's bounding box intersects the input viewport.
     - If intersecting, adds subdivision to results object.
3. If `Sub_level` === 2 (country supports second-level subdivisions):
   - For each intersecting first-level subdivision:
     - Loads its **second-level subdivision data** from  
       `/BBOX JSON/<ISO_ALPHA3>/<ISO_ALPHA3>-<SubL1 Key>_BBOX_ESPG4326.json`
     - Iterates over all second-level subdivisions:
       - Checks if each subdivision's bounding box intersects the viewport.
       - If intersecting, adds to the results under its first-level parent.
4. Returns a structured object detailing all intersecting first-level and second-level subdivisions.

---

**Returned Object Structure:**

```js
{
  "Île-de-France": {
    subL1_num: "11", // Example: region numeric code
    subL1_id: "IDF",
    source: "BBOX",
    subL2: {
      "Paris": {
        subL2_num: "75",
        source: "BBOX"
      },
      "Yvelines": {
        subL2_num: "78",
        source: "BBOX"
      }
      // ...other intersecting second-level subdivisions
    }
  },
  // ...other intersecting first-level subdivisions
}
```

- If only first-level intersects, `subL2` may be empty or omitted.

---

**Usage Example:**

```js
const france = { ISO_ALPHA3: 'FRA', Sub_level: 2 };
const subdivisions = await bboxApi.getIntersectingSubdivisions(france, viewport);
console.log(subdivisions);
// Example result:
// {
//   "Île-de-France": {
//     subL1_num: "11",
//     subL1_id: "IDF",
//     source: "BBOX",
//     subL2: {
//       "Paris": { subL2_num: "75", source: "BBOX" },
//       ...
//     }
//   },
//   ...
// }
```

---

**Notes:**

- If `Sub_level` is `0` or not present, returns an empty object.
- For subdivision data format details, see [File and Folder Convention Summary](#file-and-folder-convention-summary).
- This function **does not** perform high-precision GeoJSON intersections (unlike the USA-specific method); all checks are bounding-box based.
- Hierarchical result is organized for easy traversal, compatible with your UI or downstream analytics.
- Logs warnings if subdivision data is missing or inaccessible; errors are caught and reported to console.

**See Also:**

- [getIntersectingStatesAndCounties()](#getintersectingstatesandcountiesviewportbbox-highprecision--false-returngeojson--false)
- [getCountriesAndSubsJson()](#getcountriesandsubsjson)

---

### cleanIntersectingData(intersectingCountries)

**Purpose:**  
Prunes empty subdivisions and branches from a nested intersection result.

**Signature:**  
`cleanIntersectingData(intersectingCountries: Object): void`

**Usage Example:**

```js
bboxApi.cleanIntersectingData(results);
```

---

### fetchAndCheckGeoJsonIntersection(countyCode, subCode, subSubCode, viewportBbox, returnGeoJson = false)

**Purpose:**  
Fetches a region's GeoJSON shape given its country, subdivision, and sub-subdivision codes, and determines whether that geometry intersects the provided viewport bbox. Optionally, returns the full GeoJSON if an intersection is found.

**Signature:**  
`fetchAndCheckGeoJsonIntersection(countyCode: string, subCode: string, subSubCode: string, viewportBbox: Object, returnGeoJson?: boolean): Promise<boolean|Object>`

**Parameters:**

- `countyCode` (string):
  - ISO code for the country (e.g., `"USA"`).
- `subCode` (string):
  - ISO/state/province code, e.g. `"TX"` for Texas.
- `subSubCode` (string):
  - Numeric code or ID for the third-level subdivision (e.g. FIPS code), e.g. `"201"` for Harris County.
- `viewportBbox` (Object):
  - The bounding box you want to intersect against.
  - Format: `{ minLon: number, minLat: number, maxLon: number, maxLat: number }`
- `returnGeoJson` (boolean, default=false):
  - If `true`, returns the full GeoJSON for the intersecting region; otherwise just returns `true`/`false`.

---

**Process Overview:**

1. **Constructs the URL** to the GeoJSON file using the codes, in the form:  
   `GEOJSON/<countyCode>/<subCode>/<countyCode>-<subCode>-<subSubCode>_EPSG4326.geojson`
2. **Fetches the GeoJSON data** for the requested region via cached HTTP request.
3. **Defines the viewport** as a polygon using the four corners of the bounding box:
   - `[ [minLon, minLat], [minLon, maxLat], [maxLon, maxLat], [maxLon, minLat], [minLon, minLat] ]`
4. **Iterates through each feature** in the GeoJSON:
   - For Polygon: checks intersection between the polygon and the viewport polygon.
   - For MultiPolygon: checks intersection for every component polygon.
   - If any intersect, returns `true` (or the full GeoJSON if `returnGeoJson` is true).
5. **Returns:**
   - `true` if intersection is found (`returnGeoJson === false`),
   - the full GeoJSON object (`returnGeoJson === true`),
   - or `false` if no intersection is found.
6. **Handles geometry types:** Only supports `"Polygon"` and `"MultiPolygon"`; logs a warning for others.
7. **Error Handling:** Logs and returns `false` if any error occurs (network issue, parse error, etc).

---

**Usage Example:**

```js
const viewport = { minLon: -96.0, minLat: 29.5, maxLon: -94.9, maxLat: 30.2 };
// Returns true if Harris County intersects the viewport
const intersects = await bboxApi.fetchAndCheckGeoJsonIntersection('USA', 'TX', '201', viewport);

// Returns the full GeoJSON FeatureCollection if intersecting and returnGeoJson=true
const geojsonData = await bboxApi.fetchAndCheckGeoJsonIntersection('USA', 'TX', '201', viewport, true);
```

---

**Example GeoJSON File:**  
A file such as `GEOJSON/USA/TX/USA-TX-201_EPSG4326.geojson` contains a standard GeoJSON FeatureCollection:

```json
{
  "type": "FeatureCollection",
  "name": "US-TX-201_EPSG4326.geojson",
  "crs": { "type": "name", "properties": { "name": "urn:ogc:def:crs:OGC:1.3:CRS84" } },
  "features": [
    {
      "type": "Feature",
      "properties": { "STATEFP": "48", "COUNTYFP": "201", "NAME": "Harris" },
      "bbox": [-95.960733, 29.497297, -94.908492, 30.170606],
      "geometry": {
        "type": "MultiPolygon",
        "coordinates": [
          [
            [
              [-94.978845, 29.676731],
              ... // (thousands of coordinates omitted)
              [-94.978845, 29.676731]
            ]
          ]
        ]
      }
    }
  ]
}
```

---

**Notes:**

- This method is used internally for “high precision” spatial checks in other APIs like `getIntersectingStatesAndCounties`.
- Does not currently support GeoJSON geometry types other than `Polygon`/`MultiPolygon`.
- Intersection determination uses polygon edge+containment (see internal `hasIntersection()`).
- Returns `false` and logs an error if fetch or format fails.

---

**See Also:**

- [getIntersectingStatesAndCounties()](#getintersectingstatesandcountiesviewportbbox-highprecision--false-returngeojson--false)

---

### Master method: `whatsInView(viewportBbox, highPrecision = false, returnGeoJson = false)`

**Purpose:**  
This is the primary method for determining which countries and their subdivisions are visible within a specific viewport bounding box, with optional high-precision (GeoJSON) intersection logic for USA counties and sub-counties.

**Signature:**  
`whatsInView(viewportBbox: Object, highPrecision?: boolean, returnGeoJson?: boolean): Promise<Object>`

**Parameters:**

- `viewportBbox` (Object):  
  The bounding box representing the current map area.  
  Format: `{ minLon: number, minLat: number, maxLon: number, maxLat: number }`
- `highPrecision` (boolean, default = false):  
  If true, for supported regions (USA counties), intersection is refined using full GeoJSON shapes, not just bounding boxes.
- `returnGeoJson` (boolean, default = false):  
  If true (and highPrecision is used), attaches intersecting region GeoJSON data to the results for visualization.

---

**Process Overview:**

1. **Fetches countries intersecting the viewport**  
   via [`getIntersectingCountries()`](#getintersectingcountriesviewportbbox).
2. **For each country in view:**
   - If the country has no subdivisions (`Sub_level: 0`), returns minimal metadata.
   - If the country is the USA:
     - Uses [`getIntersectingStatesAndCounties()`](#getintersectingstatesandcountiesviewportbbox-highprecision--false-returngeojson--false)  
       to get a detailed region/county/sub-county tree, with optional high precision.
   - For other countries:
     - Uses [`getIntersectingSubdivisions()`](#getintersectingsubdivisionscountryobj-viewportbbox)  
       to get all intersecting subdivisions by level (bbox-based).
3. **Cleans up and prunes empty intersections**  
   using [`cleanIntersectingData()`](#cleanintersectingdataintersectingcountries) for a concise tree.
4. **Returns a single object** mapping country names to all visible subdivisions or minimal info.

---

**Returned Object Structure:**

```js
{
  "United States": {
    ISO_ALPHA2: "US",
    ISO_ALPHA3: "USA",
    Sub_level: 3,
    subL1: {
      "Texas": { ... } // See getIntersectingStatesAndCounties structure
    }
  },
  "France": {
    ISO_ALPHA2: "FR",
    ISO_ALPHA3: "FRA",
    Sub_level: 2,
    subL1: {
      "Île-de-France": {
        subL1_num: "11",
        subL1_id: "IDF",
        // etc...
      }
    }
  }
  // ...other intersecting countries
}
```

- If a country supports deeper administrative levels, those appear in the nested structure.
- If a country only supports `Sub_level: 0` (no subdivisions), `.subL1` is empty.

---

**Usage Example:**

```js
const bboxApi = new wmeGisLBBOX();
const viewport = { minLon: -80, minLat: 25, maxLon: -78, maxLat: 28 };
// Find all countries and subdivisions in view using bounding boxes
const results = await bboxApi.whatsInView(viewport);
// Find with high precision (e.g., for US counties/cities) and optional GeoJSON:
const highPrecise = await bboxApi.whatsInView(viewport, true, false);
const geojsonResults = await bboxApi.whatsInView(viewport, true, true);

console.log(results);
console.log(highPrecise);
console.log(geojsonResults);
```

---

**Notes:**

- **Performance:**  
  With `highPrecision`, fetching and computing may take longer for the USA due to additional network GeoJSON reads.
- For **non-USA countries**, all intersections are bbox-based and go as deep as available subdivision levels.
- If the viewport doesn’t intersect any country, the result will be an empty object.
- The structure is suitable for directly powering UI overlays or downstream analytics.

**See Also:**

- [getIntersectingStatesAndCounties()](#getintersectingstatesandcountiesviewportbbox-highprecision--false-returngeojson--false)
- [getIntersectingSubdivisions()](#getintersectingsubdivisionscountryobj-viewportbbox)
- [cleanIntersectingData()](#cleanintersectingdataintersectingcountries)

---

## Utility Functions (Internal)

Used by the main prototype; rarely called directly. Documented for completeness.

---

### checkIntersection(bbox1, bbox2)

Checks if two bbox objects intersect (with antimeridian wrap consideration).

**Signature:**  
`checkIntersection(bbox1: {minLat, maxLat, minLon, maxLon}, bbox2: {minLat, maxLat, minLon, maxLon}): boolean`

**Usage Example:**

```js
const overlaps = checkIntersection({minLat: ..., maxLat: ..., minLon: ..., maxLon: ...}, {...});
```

---

### isPointInPolygon(point, vs)

Ray-casting algorithm to determine if a point is inside a polygon.

**Signature:**  
`isPointInPolygon(point: [x, y], vs: Array<[x, y]>): boolean`

---

### segmentIntersection(p1, p2, q1, q2)

Returns point of intersection between two line segments, or null.

**Signature:**  
`segmentIntersection(p1: [x, y], p2: [x, y], q1: [x, y], q2: [x, y]): [x, y] | null`

---

### hasIntersection(polygon1, polygon2)

Checks for intersection between two polygons (via edge intersection and point containment).

**Signature:**  
`hasIntersection(polygon1: Array<[x, y]>, polygon2: Array<[x, y]>): boolean`

---

## Examples

### Find What Regions are in View

```js
const bboxApi = new wmeGisLBBOX();
const viewport = { minLon: -80, minLat: 25, maxLon: -78, maxLat: 28 };

// Basic: Countries and first-level subdivisions
const results = await bboxApi.whatsInView(viewport);
console.log(results);

// With high precision GeoJSON checks
const preciseResults = await bboxApi.whatsInView(viewport, true, false);
console.log(preciseResults);
```

---

### Fetch Counties Intersecting with Viewport

```js
const texasViewport = { minLon: -102, minLat: 29, maxLon: -94, maxLat: 36 };
const counties = await bboxApi.getIntersectingStatesAndCounties(texasViewport);
console.log(counties);
```

---

## Error Handling

- All async methods use try/catch for network errors; methods log warnings and return empty objects or arrays when fetch fails.
- If the data is missing or corrupt, subdivision methods log and skip hierarchy appropriately.
- GeoJSON intersection methods log issues, and unsupported geometry types generate warnings.

---

## License

This project is licensed under the MIT License. See LICENSE for details.

---

## Additional Notes

- All fetches use `GM_xmlhttpRequest` for cross-domain capability (userscript engines).
- Cache is per-instance; for persistent cache between page reloads, expand to use localStorage.
- All referenced JSON files (for country codes, names, and subdivision structures) are stored in this repository under the relevant folder, and they are also hosted automatically via GitHub Pages (github.io) for live data fetching.
- If you need to reference or update these files, see the appropriate folder in the repo.

---

This documentation aims to provide clarity for integrators, maintainers, and contributors on the internal APIs and extension points for wmeGisLBBOX.

---

# WME whatsInView (Script Plug-in) Overview

The `whatsInView` function is the core user-facing feature of this userscript, providing interactive map-based insights by identifying all countries and subdivisions currently visible within the Waze Map Editor (WME) viewport.

**Key Features:**

- **SDK Integration:**  
  Interfaces with the WME SDK to access the current map extent and other map functionalities, initializing only after all necessary dependencies (WME SDK, WazeWrap, etc.) are fully loaded.

- **User Interface Setup:**  
  Dynamically creates and registers a custom sidebar tab in the WME, displaying script details, attribution, and interactive options for users.

- **Popup System:**  
  Presents a popup or sidebar content labeled "What's in View?"—showing detailed geographic information about all regions, countries, and subdivisions currently visible on the map.

- **Real-time Updates:**  
  Listens for map movement, zoom, or rotation events and automatically refreshes the displayed data whenever the viewport changes, ensuring that region and subdivision information is always current.

- **Precision and Debug Controls:**  
  UI options allow users to enable/disable high-precision (GeoJSON geometry) processing (for more accurate region detection) and toggle debug logging as needed.

- **Efficient Resource Handling:**  
  Ensures all required libraries and services are ready before operating, minimizing unnecessary loading or network requests.

- **Extensibility:**  
  The modular design makes it easy to augment the information shown, integrate with other WME plugins, or add additional region-based features.

**Installation:**  
You can install this script from:  
<https://WazeDev.github.io/wmeGisLBBOX/whatsInView.js>

**How It Works:**

1. On script load, waits for the WME SDK and WazeWrap to become ready.
2. Initializes the sidebar and popup, and attaches event listeners to the map.
3. On any map movement (pan, zoom, etc.), calls the internal `.whatsInView()` API, passing the current viewport.
4. Displays a human-friendly summary of all countries and subdivisions in view—optionally including fine-grained results with high-precision geometry checks.
5. Updates instantly and automatically as the map changes.

---

With this plug-in, WME editors gain instant, accurate feedback about the administrative regions present in any area they're viewing—greatly improving context for map edits, reporting, or research.

---
