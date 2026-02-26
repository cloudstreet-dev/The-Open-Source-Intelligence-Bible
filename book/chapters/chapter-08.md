# Chapter 8: Geospatial Intelligence

## Learning Objectives

By the end of this chapter, you will be able to:
- Apply geospatial methods to OSINT investigations using commercially available tools
- Geolocate images and videos using systematic visual analysis methodology
- Exploit satellite imagery, AIS, and ADS-B data for investigative purposes
- Build automated geospatial analysis pipelines
- Integrate geospatial findings with other OSINT data streams
- Apply AI-assisted image and geolocation analysis tools

---

## 8.1 The Geospatial Revolution in OSINT

The convergence of commercial satellite imagery, proliferating GPS-enabled devices, AIS/ADS-B tracking systems, and location-enabled social media has created an unprecedented capability for geospatial intelligence that was, a decade ago, the exclusive domain of national intelligence agencies.

Consider what is now commercially available to any investigator:
- Daily sub-meter satellite imagery of anywhere on Earth
- Real-time global maritime vessel tracking
- Real-time global commercial aviation tracking
- Historical imagery dating back decades at many locations
- Crowdsourced ground-level imagery of virtually every populated location

When combined with the geolocation data embedded in social media posts and the analytical methods for extracting location from photographic evidence, geospatial intelligence (GEOINT) has become a critical investigative layer.

The Bellingcat investigation that verified Russian GRU officers were present in Salisbury, UK, before the Novichok poisoning — using only open satellite imagery, hotel booking sites, and geotagged social media — illustrates the power now available to civilian investigators.

---

## 8.2 Systematic Visual Geolocation

Visual geolocation — determining where a photograph or video was taken by analyzing its content — is a systematic investigative process, not a parlor trick. It requires methodological rigor, patience, and knowledge of how to use available tools.

### The Geolocation Process

**Step 1: Establish geographic constraints**

Before examining image details, establish the broadest possible geographic constraints:

- **Language**: Text visible in the image (signs, storefronts, documents) narrows language/region
- **Driving side**: Left-hand vs. right-hand traffic narrows to specific country lists
- **Electrical infrastructure**: Power pole design, connector types, and electrical infrastructure varies by country
- **Vehicle types**: Make, model, and license plate format narrow geography
- **Architecture**: Building styles, materials, and construction methods are geographically distinctive
- **Vegetation**: Plant species, seasonal state, and vegetation patterns are geographically constrained
- **Terrain**: Mountain profiles, soil color, and terrain type narrow location

**Step 2: Identify landmark candidates**

Identify specific, searchable features:
- Visible building facades and distinctive architecture
- Business names and signage
- Street signs, intersection layouts
- Infrastructure features (bridges, radio towers, distinctive roads)
- Natural features (river bends, distinctive ridgelines, rock formations)

**Step 3: Search and verify**

Use available tools to locate identified landmarks:
- Google Street View for ground-level verification
- Google Earth for aerial perspective matching
- Bing Maps (different imagery date/coverage)
- OpenStreetMap for business/feature identification
- Yandex Maps (often better coverage in Russia and Central Asia)
- Baidu Maps (best coverage in China)
- Apple Maps (independent imagery source)

**Step 4: Confirmatory analysis**

Verify the proposed location through multiple independent elements:
- Shadow direction and length (solar position analysis)
- Multiple visible features that all match the proposed location
- Internal consistency between image details and proposed location

### Solar Analysis for Temporal Geolocation

Shadow direction and length constrain both the time of day and the season when a photograph was taken. Combined with geographic location, shadow analysis can narrow a photograph's time of capture to within hours.

**Tools**:
- **SunCalc.org**: Enter a location and date to see sun position throughout the day; compare to observed shadow direction and length
- **ShadowCalculator.com**: More precise shadow modeling tool
- **Timeanddate.com**: Sun position calculator with sunrise/sunset data

```python
from astral import LocationInfo
from astral.sun import sun
from datetime import datetime, timezone
import math

def solar_position_at(latitude, longitude, datetime_utc):
    """Calculate solar azimuth and elevation at a given location/time"""
    from astral import LocationInfo
    from astral.sun import sun, azimuth as solar_azimuth, elevation as solar_elevation

    loc = LocationInfo(
        name="Investigation Location",
        region="",
        timezone="UTC",
        latitude=latitude,
        longitude=longitude
    )

    s = sun(loc.observer, date=datetime_utc.date())
    az = solar_azimuth(loc.observer, datetime_utc)
    el = solar_elevation(loc.observer, datetime_utc)

    return {
        'azimuth': az,
        'elevation': el,
        'sunrise': str(s['sunrise']),
        'sunset': str(s['sunset']),
        'noon': str(s['noon']),
    }

def estimate_photo_time(latitude, longitude, shadow_direction_degrees, target_date):
    """
    Estimate when a photo was taken based on shadow direction
    shadow_direction_degrees: direction the shadow points (0=North, 90=East, etc.)
    """
    from astral.sun import azimuth as solar_azimuth

    loc_info = LocationInfo(
        latitude=latitude,
        longitude=longitude,
        timezone="UTC"
    )

    # Shadow direction is opposite solar azimuth
    solar_azimuth_from_shadow = (shadow_direction_degrees + 180) % 360

    # Check each hour of the day
    possible_times = []
    for hour in range(6, 20):  # Daylight hours
        for minute in [0, 15, 30, 45]:
            dt = datetime(target_date.year, target_date.month, target_date.day,
                         hour, minute, tzinfo=timezone.utc)
            az = solar_azimuth(loc_info.observer, dt)

            if abs(az - solar_azimuth_from_shadow) < 5:  # Within 5 degrees
                possible_times.append({
                    'time_utc': str(dt),
                    'solar_azimuth': az,
                    'match_quality': abs(az - solar_azimuth_from_shadow)
                })

    return sorted(possible_times, key=lambda x: x['match_quality'])
```

---

## 8.3 Satellite Imagery Analysis

### Commercial Imagery Sources

**Planet Labs**
- Daily global coverage at 3-5 meter resolution (PlanetScope)
- 50cm resolution on specific tasked areas (SkySat)
- Subscription required; research access programs exist
- Excellent for monitoring temporal changes at fixed locations
- Python SDK available

**Maxar Technologies (WorldView)**
- 30cm resolution (highest quality commercial imagery)
- Extensive archive going back to 1999
- Used by Google Earth and Google Maps
- Point-in-time tasking available
- Enterprise pricing

**Sentinel (Copernicus/ESA)**
- Free, open access imagery
- 10-60m resolution depending on sensor
- 5-day global revisit
- Available via Copernicus Open Access Hub and Google Earth Engine

**Google Earth Pro (free desktop application)**
- Historical imagery dating back to the 1970s at some locations
- Excellent interface for temporal comparison
- Cannot be used for commercial purposes without licensing

```python
# Working with Sentinel-2 imagery via SentinelHub
from sentinelhub import (
    SHConfig, BBox, CRS, DataCollection,
    SentinelHubRequest, MimeType, MosaickingOrder
)
from datetime import date
import numpy as np
import matplotlib.pyplot as plt

def get_sentinel_imagery(bbox_coords, date_from, date_to, resolution=10):
    """
    Retrieve Sentinel-2 imagery for an area of interest
    bbox_coords: (min_lon, min_lat, max_lon, max_lat)
    """
    config = SHConfig()
    config.sh_client_id = 'YOUR_CLIENT_ID'
    config.sh_client_secret = 'YOUR_CLIENT_SECRET'

    bbox = BBox(bbox=bbox_coords, crs=CRS.WGS84)

    request = SentinelHubRequest(
        evalscript="""
        //VERSION=3
        function setup() {
          return { input: ["B04", "B03", "B02"], output: { bands: 3 } };
        }
        function evaluatePixel(sample) {
          return [2.5*sample.B04, 2.5*sample.B03, 2.5*sample.B02];
        }
        """,
        input_data=[
            SentinelHubRequest.input_data(
                data_collection=DataCollection.SENTINEL2_L2A,
                time_interval=(date_from, date_to),
                mosaicking_order=MosaickingOrder.LEAST_CC,
            )
        ],
        responses=[SentinelHubRequest.output_response('default', MimeType.PNG)],
        bbox=bbox,
        size=(1024, 1024),
        config=config,
    )

    images = request.get_data()
    return images[0] if images else None

def detect_changes(image_before, image_after, threshold=30):
    """Simple change detection between two imagery dates"""
    diff = np.abs(image_before.astype(int) - image_after.astype(int))
    change_mask = diff.mean(axis=2) > threshold

    change_percentage = change_mask.mean() * 100
    return change_mask, change_percentage
```

### Temporal Analysis: Before/After Comparison

The most powerful geospatial analytical technique is temporal comparison — comparing imagery from before and after a significant event to document change. This methodology:

- Verifies or disproves claims about what happened at a location
- Documents construction, destruction, or modification of infrastructure
- Tracks movement of military equipment, vehicles, or people
- Establishes timeline of activities at a location

**Investigative applications**:
- Conflict zone monitoring (documenting military deployments, destruction of civilian infrastructure)
- Environmental monitoring (illegal deforestation, mining operations, pollution)
- Sanctions enforcement (monitoring sanctioned industrial facilities)
- Construction monitoring (verifying or investigating development projects)
- Verification of eyewitness accounts about location-specific events

---

## 8.4 Maritime Intelligence: AIS Tracking

The Automatic Identification System (AIS) is an international maritime standard that requires most commercial vessels to broadcast their position, course, speed, and identity at regular intervals. This data is received by terrestrial stations and satellite receivers and aggregated by commercial services.

### AIS Data Fields

AIS transmissions include:
- **MMSI number**: Unique vessel identifier (Maritime Mobile Service Identity)
- **Vessel name and callsign**
- **IMO number**: International Maritime Organization registration number (permanent)
- **Position coordinates** (GPS accuracy)
- **Speed over ground** and **course over ground**
- **Destination and ETA** (self-reported, not verified)
- **Vessel type** (container, tanker, passenger, etc.)
- **Dimensions**
- **Navigation status** (underway, anchored, etc.)

### AIS Data Sources

**MarineTraffic**: The largest public AIS aggregator. Free tier provides limited historical access; paid tiers enable bulk data queries and extended history.

**VesselFinder**: Alternative to MarineTraffic with similar coverage.

**Fleetmon**: Another AIS aggregator with API access.

**UN Global Platform**: Some AIS data available through UN research initiatives.

**Raw AIS**: Open-source AIS receivers can collect local VHF transmissions. Satellite AIS (S-AIS) extends coverage to open ocean.

### AIS Investigation Methodology

```python
import requests
from datetime import datetime, timedelta

class MarineTrafficAPI:
    """MarineTraffic API client for vessel intelligence"""

    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://services.marinetraffic.com/api"

    def get_vessel_track(self, mmsi, from_date, to_date):
        """Get historical track for a vessel"""
        endpoint = f"{self.base_url}/exportvesseltrack/v:5/{self.api_key}"
        params = {
            'v': 5,
            'mmsi': mmsi,
            'fromdate': from_date,
            'todate': to_date,
            'protocol': 'json'
        }

        response = requests.get(endpoint, params=params)
        return response.json() if response.status_code == 200 else {}

    def get_vessel_photos(self, mmsi):
        """Get photos associated with a vessel"""
        endpoint = f"{self.base_url}/getphotos/v:1/{self.api_key}"
        response = requests.get(endpoint, params={'mmsi': mmsi})
        return response.json() if response.status_code == 200 else {}

    def search_vessels_in_area(self, lat, lon, radius_km, vessel_type=None):
        """Find vessels in a geographic area"""
        endpoint = f"{self.base_url}/getvessel/v:8/{self.api_key}"
        params = {
            'lat': lat,
            'lon': lon,
            'radius': radius_km * 1000,  # API expects meters
        }
        if vessel_type:
            params['type'] = vessel_type

        response = requests.get(endpoint, params=params)
        return response.json() if response.status_code == 200 else {}

def investigate_vessel(mmsi, investigation_window_days=30):
    """Build vessel intelligence profile"""
    # Get vessel information
    response = requests.get(
        f"https://www.marinetraffic.com/getData/get_data_json_4/z:1/X:0/Y:0/station:0",
        params={'mmsi': mmsi}
    )

    profile = {
        'mmsi': mmsi,
        'investigation_note': 'Use MarineTraffic API with valid key for production use'
    }

    return profile
```

### AIS Gap Analysis

A critical investigative technique is identifying when a vessel's AIS transponder was switched off — known as "going dark." AIS gaps can indicate:

- Entering waters where AIS transmission is not satellite-monitored (coverage gaps)
- Deliberate transponder deactivation (common for sanctions evasion)
- Technical failure (less common on modern vessels)
- Entry into a port without coastal AIS coverage

AIS gaps combined with port calls, cargo records, and geographic analysis have been used to trace sanctions-evading oil shipments, document illegal fishing, and track suspected criminal vessels.

---

## 8.5 Aviation Intelligence: ADS-B Tracking

ADS-B (Automatic Dependent Surveillance-Broadcast) is the aviation equivalent of AIS. Commercial aircraft broadcast their position, altitude, speed, and identity to ground stations and satellite receivers. This data is aggregated by services like FlightAware and Flightradar24.

### ADS-B Data Sources

**Flightradar24**: The most widely used real-time flight tracking service. Historical data available via API for a fee.

**FlightAware**: Strong U.S. coverage with API access. Historical flight data available.

**ADS-B Exchange**: Community-maintained receiver network that does NOT filter military or private jets. The only source for aircraft that other services hide (often government or sensitive private operators).

**OpenSky Network**: Academic ADS-B database with free research access and historical data going back several years.

### Aviation Intelligence Use Cases

**Private aircraft investigation**: Tracking private jet registrations to identify ownership and movement patterns. The FAA N-number registry links tail numbers to registered owners (though some registrations use management companies or trusts as shields).

**VIP movement tracking**: High-profile individuals often travel via private aircraft. Tracking private jet movements can establish presence at locations.

**Government aircraft surveillance**: Covert government aircraft operations have been exposed by ADS-B tracking of aircraft making unusual patterns suggesting surveillance flights.

**Sanctions evasion**: Like maritime AIS analysis, ADS-B tracking can reveal aviation movement that contradicts sanctions compliance.

```python
import requests
import json
from datetime import datetime

class OpenSkyAPI:
    """OpenSky Network API client"""

    BASE_URL = "https://opensky-network.org/api"

    def __init__(self, username=None, password=None):
        self.auth = (username, password) if username else None

    def get_states(self, time=None, icao24=None, bbox=None):
        """Get current or historical aircraft states"""
        params = {}
        if time:
            params['time'] = time
        if icao24:
            params['icao24'] = icao24 if isinstance(icao24, str) else ','.join(icao24)
        if bbox:  # (min_lat, max_lat, min_lon, max_lon)
            params.update({
                'lamin': bbox[0], 'lamax': bbox[1],
                'lomin': bbox[2], 'lomax': bbox[3]
            })

        response = requests.get(
            f"{self.BASE_URL}/states/all",
            params=params,
            auth=self.auth
        )

        if response.status_code == 200:
            data = response.json()
            states = []
            for s in (data.get('states') or []):
                states.append({
                    'icao24': s[0],
                    'callsign': s[1].strip() if s[1] else None,
                    'origin_country': s[2],
                    'last_contact': datetime.fromtimestamp(s[3]).isoformat() if s[3] else None,
                    'longitude': s[5],
                    'latitude': s[6],
                    'altitude_m': s[7],
                    'on_ground': s[8],
                    'velocity_ms': s[9],
                    'heading': s[10],
                    'squawk': s[14],
                })
            return states
        return []

    def get_flights_by_aircraft(self, icao24, begin_timestamp, end_timestamp):
        """Get flight history for a specific aircraft"""
        response = requests.get(
            f"{self.BASE_URL}/flights/aircraft",
            params={
                'icao24': icao24,
                'begin': begin_timestamp,
                'end': end_timestamp
            },
            auth=self.auth
        )
        return response.json() if response.status_code == 200 else []

def lookup_faa_registration(n_number):
    """Look up FAA aircraft registration by N-number"""
    # FAA Aircraft Registry (ReleasedAircraftInformation on FAA.gov)
    # API endpoint for individual lookups
    clean_n = n_number.upper().replace('N', '', 1) if n_number.startswith('N') else n_number

    response = requests.get(
        f"https://api.faa.gov/aircraft/{n_number}",
        headers={'apikey': 'your_faa_api_key'}  # FAA API registration required
    )

    # Alternative: scrape the publicly accessible search
    alt_response = requests.get(
        "https://registry.faa.gov/aircraftinquiry/Search/NNumberInquiry",
        params={'nNumberTxt': n_number}
    )

    return alt_response.text  # Parse HTML for registration data
```

---

## 8.6 Location Data from Social Media and IoT

### GPS-Enabled Social Media

Many social media posts contain geographic data either explicitly (location tags, check-ins) or implicitly (EXIF metadata in uploaded images):

**Extracting location data at scale**:

```python
import tweepy
import json
from geopy.geocoders import Nominatim
from geopy.distance import distance

def collect_geotagged_tweets(query, bbox, max_results=100):
    """
    Collect geotagged tweets within a bounding box
    bbox: [southwest_lon, southwest_lat, northeast_lon, northeast_lat]
    """
    # Note: Twitter API v2 requires appropriate access tier for geo search
    client = tweepy.Client(bearer_token='YOUR_BEARER_TOKEN')

    # Build query with geo filter
    geo_query = f"{query} bounding_box:[{' '.join(map(str, bbox))}]"

    tweets = client.search_recent_tweets(
        query=geo_query,
        tweet_fields=['created_at', 'geo', 'author_id', 'entities'],
        expansions=['geo.place_id', 'author_id'],
        place_fields=['contained_within', 'country', 'full_name', 'geo', 'name', 'place_type'],
        max_results=min(max_results, 100)
    )

    geolocated = []
    if tweets.data:
        places = {p.id: p for p in (tweets.includes.get('places') or [])}
        for tweet in tweets.data:
            if tweet.geo:
                place = places.get(tweet.geo.get('place_id'))
                geolocated.append({
                    'tweet_id': tweet.id,
                    'text': tweet.text,
                    'created_at': str(tweet.created_at),
                    'coordinates': tweet.geo.get('coordinates'),
                    'place_name': place.full_name if place else None,
                    'place_type': place.place_type if place else None,
                })

    return geolocated

def analyze_location_patterns(geolocated_posts):
    """Analyze spatial patterns in geolocated content"""
    from collections import Counter
    import statistics

    coordinates = [
        (p['coordinates']['coordinates'][1], p['coordinates']['coordinates'][0])
        for p in geolocated_posts
        if p.get('coordinates') and p['coordinates'].get('coordinates')
    ]

    if not coordinates:
        return {}

    lats = [c[0] for c in coordinates]
    lons = [c[1] for c in coordinates]

    return {
        'total_posts': len(coordinates),
        'centroid': (statistics.mean(lats), statistics.mean(lons)),
        'lat_range': (min(lats), max(lats)),
        'lon_range': (min(lons), max(lons)),
        'geographic_spread_km': distance(
            (min(lats), min(lons)),
            (max(lats), max(lons))
        ).km
    }
```

### Fitness App Data Leakage

Strava's 2018 global heat map inadvertently revealed military base perimeters and patrol routes in conflict zones because soldiers used fitness trackers during duty. This is a canonical example of how aggregate location data from IoT-class devices reveals patterns invisible in individual data points.

For investigators, this suggests:
- Fitness app data can reveal routine patterns (home address inference, workplace location)
- IoT device data aggregated at scale reveals operational patterns
- Public "challenges" and route-sharing features create unintended location disclosure

---

## 8.7 Building a Geospatial OSINT Workflow

Integrating geospatial data with other OSINT sources creates a multi-layer picture:

```python
from dataclasses import dataclass, field
from typing import List, Dict, Tuple, Optional
import json

@dataclass
class GeospatialEvent:
    """A confirmed or probable event at a geographic location"""
    location: Tuple[float, float]  # (lat, lon)
    location_name: str
    timestamp: str
    event_type: str  # 'sighting', 'activity', 'structure', 'vessel', 'aircraft'
    source: str
    confidence: str  # 'confirmed', 'probable', 'possible'
    evidence: List[str] = field(default_factory=list)
    notes: str = ""

class GeospatialInvestigation:
    """Manages geospatial data in an OSINT investigation"""

    def __init__(self, subject_name: str):
        self.subject = subject_name
        self.events: List[GeospatialEvent] = []
        self.locations_of_interest: List[Dict] = []

    def add_event(self, event: GeospatialEvent):
        self.events.append(event)

    def get_location_timeline(self):
        """Return events sorted by timestamp"""
        return sorted(self.events, key=lambda e: e.timestamp)

    def find_co-location_patterns(self, time_window_hours=24):
        """Identify events that occurred in the same area within a time window"""
        from geopy.distance import distance
        from datetime import datetime, timedelta

        patterns = []
        for i, event1 in enumerate(self.events):
            for event2 in self.events[i+1:]:
                try:
                    dist = distance(event1.location, event2.location).km
                    dt1 = datetime.fromisoformat(event1.timestamp)
                    dt2 = datetime.fromisoformat(event2.timestamp)
                    time_diff = abs((dt2 - dt1).total_seconds() / 3600)

                    if dist < 1.0 and time_diff < time_window_hours:
                        patterns.append({
                            'event1': event1,
                            'event2': event2,
                            'distance_km': round(dist, 3),
                            'time_diff_hours': round(time_diff, 2)
                        })
                except Exception:
                    continue

        return patterns

    def export_geojson(self):
        """Export events as GeoJSON for visualization"""
        features = []
        for event in self.events:
            features.append({
                "type": "Feature",
                "geometry": {
                    "type": "Point",
                    "coordinates": [event.location[1], event.location[0]]
                },
                "properties": {
                    "name": event.location_name,
                    "timestamp": event.timestamp,
                    "type": event.event_type,
                    "source": event.source,
                    "confidence": event.confidence,
                    "notes": event.notes
                }
            })

        return json.dumps({
            "type": "FeatureCollection",
            "name": f"OSINT Investigation: {self.subject}",
            "features": features
        }, indent=2)
```

---

## 8.8 AI-Assisted Geospatial Analysis

AI is transforming geospatial OSINT in several ways:

**Automatic landmark recognition**: Computer vision models can identify landmarks, building types, and geographic features in images, accelerating the visual geolocation process.

**Change detection**: ML models applied to satellite imagery can automatically identify changes between imagery dates, flagging areas for human review.

**Object detection in satellite imagery**: Models trained on satellite imagery can detect and count vehicles, structures, vessels, and military equipment.

**Language and text extraction**: OCR models extract visible text from images, and NLP models identify the language and geographic relevance of extracted text.

```python
from transformers import pipeline
import anthropic

def ai_assisted_geolocation(image_path, description=None):
    """
    Use multimodal AI to assist with image geolocation
    """
    import base64

    with open(image_path, 'rb') as f:
        image_data = base64.b64encode(f.read()).decode('utf-8')

    client = anthropic.Anthropic()

    # Use Claude's vision capability for geolocation analysis
    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": "image/jpeg",
                            "data": image_data,
                        },
                    },
                    {
                        "type": "text",
                        "text": """Analyze this image for geographic indicators to help with location identification.

Please identify:
1. Any visible text (signs, labels, logos)
2. Language of any text visible
3. Vehicle types and any visible license plate patterns
4. Architectural style and building materials
5. Vegetation types
6. Infrastructure features (road markings, electrical infrastructure, etc.)
7. Any distinctive landmarks
8. Possible geographic region based on all visible indicators

Do not speculate beyond what is visible. Note confidence level for each indicator."""
                    }
                ],
            }
        ],
    )

    return {
        'ai_analysis': message.content[0].text,
        'image_path': image_path,
        'description': description
    }
```

---

## Summary

Geospatial intelligence has been democratized by commercial satellite imagery, maritime AIS and aviation ADS-B tracking, GPS-enabled social media, and AI-assisted image analysis. What was once available only to intelligence agencies is now accessible to any investigator with commercial subscriptions and appropriate methodology.

Systematic visual geolocation uses structured analysis of image content — language, infrastructure, architecture, vegetation, shadows — to determine where photographs were taken. Solar analysis adds temporal constraints. Satellite imagery enables before/after comparison that documents events at specific locations.

Maritime AIS and aviation ADS-B data enable tracking of commercial vessels and aircraft globally. AIS gap analysis reveals potential sanctions evasion. Aviation transponder data tracking has exposed covert government surveillance programs and private jet movements.

Building integrated geospatial investigation workflows — combining imagery, tracking data, social media geolocation, and AI analysis — enables sophisticated situational intelligence that no single source could provide.

---

## Common Mistakes and Pitfalls

- **Single-indicator geolocation**: Identifying a location from one visual clue without confirmatory analysis
- **AIS trust without verification**: AIS data can be spoofed; confirmed vessel identity requires multiple verification methods
- **Imagery age confusion**: Satellite imagery may be months or years old; always verify imagery date before drawing conclusions about current conditions
- **Overreliance on one mapping service**: Different services have different imagery dates and coverage — always cross-reference
- **Shadow analysis calculation errors**: Incorrect shadow/sun relationship assumptions lead to wrong temporal conclusions
- **Missing ADS-B blind spots**: Not all aircraft transmit ADS-B; military and some government aircraft deliberately don't

---

## Further Reading

- Bellingcat — Geolocation Investigation Guide (available on bellingcat.com)
- Benjamin Strick, "A Guide to Open Source Geolocation" — Bellingcat
- C4ADS Vessel Tracking Methodology
- Forensic Architecture methodology — sophisticated integration of satellite, social media, and geospatial evidence
- Planet Labs API documentation for satellite imagery access
