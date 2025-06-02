# GeoIP Integration with OpenSearch - Complete Setup Guide

## Overview
This guide demonstrates how to set up IP geolocation enrichment in OpenSearch using MaxMind's GeoLite2 database and Vector.dev for log ingestion.

## Prerequisites
- OpenSearch cluster (version 2.x or later) with ingest-geoip module
- MaxMind account for GeoLite2 database access
- Vector.dev (version 0.44.0 or later)
- System permissions for file operations and service restarts

## Step 1: MaxMind GeoLite2 Setup

### 1.1 Account and License Key
1. Create a free account at [MaxMind Sign Up](https://www.maxmind.com/en/geolite2/signup)
2. Generate a license key from the MaxMind Account Portal
3. Save the license key securely for database downloads

### 1.2 Download GeoLite2 Database
```bash
# Download the GeoLite2 City database
curl -o GeoLite2-City.tar.gz \
  "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-City&license_key=YOUR_LICENSE_KEY&suffix=tar.gz"

# Extract the database
tar -xzf GeoLite2-City.tar.gz

# The extracted file structure will be: GeoLite2-City_YYYYMMDD/GeoLite2-City.mmdb
```

## Step 2: OpenSearch Configuration

### 2.1 Install GeoLite2 Database
```bash
# Create GeoIP database directory
mkdir -p /usr/share/opensearch/config/geoip-databases/

# Copy the database file
cp GeoLite2-City_*/GeoLite2-City.mmdb /usr/share/opensearch/config/geoip-databases/

# For Docker deployments
docker cp GeoLite2-City.mmdb opensearch:/usr/share/opensearch/config/geoip-databases/
```

### 2.2 Update OpenSearch Configuration
Add to `opensearch.yml`:
```yaml
ingest.geoip.database_path: /usr/share/opensearch/config/geoip-databases/
```

### 2.3 Restart OpenSearch
```bash
# Standard deployment
systemctl restart opensearch

# Docker deployment
docker restart opensearch
```

### 2.4 Verify Installation
```bash
# Check if ingest-geoip plugin is loaded
curl -X GET "http://localhost:9200/_cat/plugins?v"
```

## Step 3: Create GeoIP Ingest Pipeline

### 3.1 Enhanced GeoIP Pipeline
This pipeline handles both public and private IP addresses with proper geo_point formatting:

```bash
curl -X PUT "http://localhost:9200/_ingest/pipeline/geoip-pipeline-with-geopoint" \
-H 'Content-Type: application/json' -d'{
  "description": "Pipeline to enrich logs with GeoIP data and ensure proper geo_point format",
  "processors": [
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geo",
        "database_file": "GeoLite2-City.mmdb",
        "properties": [
          "IP",
          "COUNTRY_ISO_CODE",
          "COUNTRY_NAME", 
          "CONTINENT_NAME",
          "REGION_ISO_CODE",
          "REGION_NAME",
          "CITY_NAME",
          "TIMEZONE",
          "LOCATION"
        ],
        "ignore_missing": true,
        "ignore_failure": true
      }
    },
    {
      "script": {
        "description": "Check if GeoIP data exists, format geo_point properly, and handle private IPs",
        "source": "if (ctx.geo == null || ctx.geo.isEmpty()) { ctx.geo = [:]; ctx.geo.ip_type = '\''private'\''; ctx.geo.message = '\''GeoIP data not available - likely private IP address'\''; ctx.geo.continent_name = '\''N/A'\''; ctx.geo.country_name = '\''N/A'\''; ctx.geo.city_name = '\''N/A'\''; } else { ctx.geo.ip_type = '\''public'\''; ctx.geo.message = '\''GeoIP data enriched successfully'\''; if (ctx.geo.location != null && ctx.geo.location.lat != null && ctx.geo.location.lon != null) { ctx.geo.geo_point = [:]; ctx.geo.geo_point.lat = ctx.geo.location.lat; ctx.geo.geo_point.lon = ctx.geo.location.lon; ctx.geo.location_string = ctx.geo.location.lat + '\'','\'' + ctx.geo.location.lon; } }"
      }
    }
  ]
}'
```

## Step 4: Vector.dev Configuration

### 4.1 Install Vector.dev
```bash
# Install Vector
curl --proto '=https' --tlsv1.2 -sSfL https://sh.vector.dev | bash

# Verify installation
vector --version
```

### 4.2 Vector Pipeline Configuration
Create `vector.yaml`:

```yaml
# Vector configuration for GeoIP log ingestion
sources:
  demo_logs:
    type: generator
    format: shuffle
    lines:
      - '{"client_ip":"8.8.8.8","event":"page_view","user_agent":"Mozilla/5.0","timestamp":"{{ timestamp }}"}'
      - '{"client_ip":"1.1.1.1","event":"login","user_agent":"Chrome/91.0","timestamp":"{{ timestamp }}"}'
      - '{"client_ip":"192.168.1.100","event":"signup","user_agent":"Firefox/89.0","timestamp":"{{ timestamp }}"}'
      - '{"client_ip":"208.67.222.222","event":"purchase","user_agent":"Safari/14.1","timestamp":"{{ timestamp }}"}'
    interval: 2.0

  # Alternative: File-based source
  file_logs:
    type: file
    include: ["/path/to/logs/*.jsonl"]
    read_from: "beginning"

transforms:
  parse_and_structure:
    type: remap
    inputs: ["demo_logs"]
    source: |
      # Parse the log line if it's a string
      if is_string(.message) {
        . = parse_json!(.message)
      }
      
      # Ensure timestamp is in ISO format
      if !exists(.timestamp) {
        .timestamp = now()
      }
      
      # Add metadata
      .log_source = "vector_demo"
      .processed_at = now()

sinks:
  opensearch:
    type: opensearch
    inputs: ["parse_and_structure"]
    endpoints: ["http://localhost:9200"]
    index: "web-logs-{{ strftime(now(), "%Y-%m") }}"
    
    # Authentication (adjust as needed)
    auth:
      strategy: "basic"
      user: "admin"
      password: "your_password"
    
    # Apply GeoIP pipeline
    pipeline: "geoip-pipeline-with-geopoint"
    
    # Index template settings
    template:
      name: "web-logs-template"
      patterns: ["web-logs-*"]
      settings:
        number_of_shards: 1
        number_of_replicas: 0
      mappings:
        properties:
          client_ip:
            type: ip
          timestamp:
            type: date
          event:
            type: keyword
          geo:
            properties:
              geo_point:
                type: geo_point
              location:
                type: geo_point
              country_name:
                type: keyword
              city_name:
                type: keyword
              continent_name:
                type: keyword

# Logging configuration
api:
  enabled: true
  address: "127.0.0.1:8686"

# Enable metrics
[sinks.console_metrics]
type = "console"
inputs = ["internal_metrics"]
target = "stdout"
encoding.codec = "json"
```

### 4.3 Run Vector Pipeline
```bash
# Validate configuration
vector validate vector.yaml

# Run Vector
vector --config vector.yaml
```

## Step 5: Verification and Testing

### 5.1 Check Indexed Data
```bash
# Query the enriched logs
curl -X GET "http://localhost:9200/web-logs-*/_search?pretty" \
-H 'Content-Type: application/json' -d'{
  "query": {
    "match_all": {}
  },
  "size": 5,
  "_source": ["client_ip", "event", "geo"]
}'
```

### 5.2 Verify GeoIP Enrichment
```bash
# Search for specific geo data
curl -X GET "http://localhost:9200/web-logs-*/_search?pretty" \
-H 'Content-Type: application/json' -d'{
  "query": {
    "bool": {
      "must": [
        {"exists": {"field": "geo.geo_point"}},
        {"term": {"geo.ip_type": "public"}}
      ]
    }
  }
}'
```

### 5.3 Expected Output Structure
```json
{
  "_source": {
    "client_ip": "8.8.8.8",
    "event": "page_view",
    "timestamp": "2025-06-02T10:30:00Z",
    "geo": {
      "ip_type": "public",
      "message": "GeoIP data enriched successfully",
      "continent_name": "North America",
      "country_name": "United States",
      "country_iso_code": "US",
      "region_name": "California",
      "city_name": "Mountain View",
      "timezone": "America/Los_Angeles",
      "location": {
        "lat": 37.4056,
        "lon": -122.0775
      },
      "geo_point": {
        "lat": 37.4056,
        "lon": -122.0775
      },
      "location_string": "37.4056,-122.0775"
    }
  }
}
```

## Step 6: Index Mapping for Geo Visualization

### 6.1 Create Proper Index Template
```bash
curl -X PUT "http://localhost:9200/_index_template/web-logs-template" \
-H 'Content-Type: application/json' -d'{
  "index_patterns": ["web-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "client_ip": {"type": "ip"},
        "timestamp": {"type": "date"},
        "event": {"type": "keyword"},
        "geo": {
          "properties": {
            "geo_point": {"type": "geo_point"},
            "location": {"type": "geo_point"},
            "country_name": {"type": "keyword"},
            "city_name": {"type": "keyword"},
            "continent_name": {"type": "keyword"},
            "ip_type": {"type": "keyword"}
          }
        }
      }
    }
  }
}'
```

## Key Features

### GeoIP Pipeline Benefits
- **Comprehensive Data**: Extracts country, region, city, timezone, and coordinates
- **Error Handling**: Gracefully handles private IPs and missing data
- **Geo-Point Format**: Creates properly formatted geo_point fields for mapping
- **Dual Location Format**: Provides both object and string coordinate formats

### Vector.dev Integration
- **Real-time Processing**: Continuous log ingestion and enrichment
- **Flexible Sources**: Support for files, generators, and network sources
- **Built-in Transforms**: JSON parsing and data manipulation
- **Index Management**: Automatic template creation and time-based indexing

## Production Considerations

### Database Updates
- GeoLite2 requires updates every 30 days per EULA
- Implement automated updates using MaxMind's `geoipupdate` tool
- Monitor database freshness in production environments

### Performance Optimization
- Consider database caching for high-throughput scenarios
- Use multiple OpenSearch nodes for distributed processing
- Monitor pipeline performance and adjust batch sizes

### Legal Compliance
Include MaxMind attribution: *"This product includes GeoLite2 data created by MaxMind, available from https://www.maxmind.com."*

## Troubleshooting

### Common Issues
1. **Private IP Addresses**: Will show `ip_type: "private"` with N/A geo data
2. **Missing Database**: Verify file path and OpenSearch configuration
3. **Pipeline Errors**: Check OpenSearch logs for processor failures
4. **Mapping Conflicts**: Ensure proper index templates are applied

### Verification Commands
```bash
# Check pipeline status
curl -X GET "http://localhost:9200/_ingest/pipeline/geoip-pipeline-with-geopoint"

# Monitor Vector metrics
curl -X GET "http://127.0.0.1:8686/metrics"

# Check index health
curl -X GET "http://localhost:9200/_cat/indices/web-logs-*?v"
```

This setup provides a robust foundation for IP geolocation in OpenSearch with proper error handling and visualization-ready data formats.