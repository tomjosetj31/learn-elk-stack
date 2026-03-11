# Day 5: Kibana

## What is Kibana?

Kibana is the **web-based visualization and exploration UI** for the Elastic Stack. It lets you search, analyze, and visualize data stored in Elasticsearch indices.

Access Kibana at: `http://localhost:5601`

```
┌─────────────────────────────────────────────────────────────┐
│                    KIBANA MAIN FEATURES                      │
│                                                             │
│   Discover       → Explore & search raw data                │
│   Dashboards     → Build interactive dashboards             │
│   Visualizations → Charts, maps, tables                     │
│   Lens           → Drag-and-drop visualization editor       │
│   Canvas         → Custom pixel-perfect presentations       │
│   Maps           → Geo-spatial data visualization           │
│   Dev Tools      → Run Elasticsearch queries manually       │
│   Stack Monitoring → Monitor Elasticsearch/Kibana health   │
│   Alerting       → Set up alerts and rules                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Setting Up a Data View (Index Pattern)

Before exploring data, you need to create a **Data View** that tells Kibana which index/indices to read from.

1. Go to **Stack Management** > **Data Views**
2. Click **Create data view**
3. Enter index pattern, e.g., `logs-*`
4. Select the timestamp field (usually `@timestamp`)
5. Click **Save data view to Kibana**

---

## Discover

The **Discover** page lets you interactively explore your data.

### Key Features
- **Search bar**: KQL (Kibana Query Language) or Lucene syntax
- **Time picker**: Filter data by time range
- **Field list**: Browse all fields in the index
- **Document table**: View raw documents
- **Histogram**: See document counts over time

### KQL Examples

```kql
# Simple field match
level: "error"

# Wildcard
message: *timeout*

# Range
response_time > 1.0

# AND/OR
level: "error" AND service: "api"

# NOT
NOT level: "info"

# Parentheses for grouping
(level: "error" OR level: "warn") AND service: "api"

# Exists check
error_code: *
```

---

## Visualizations

### Types of Visualizations

| Type | Use Case |
|------|---------|
| Bar chart | Compare values across categories |
| Line chart | Trends over time |
| Pie chart | Proportions / percentages |
| Area chart | Cumulative trends over time |
| Data table | Tabular breakdown |
| Metric | Single big number (count, avg) |
| Gauge | Current value vs. threshold |
| Tag cloud | Frequency of terms |
| Heat map | Two-dimensional frequency |
| Coordinate map | Geo-spatial data |

### Creating a Visualization with Lens

1. Go to **Analytics** > **Visualize Library**
2. Click **Create visualization**
3. Choose **Lens**
4. Drag and drop fields onto the chart axes
5. Choose chart type (bar, line, pie, etc.)
6. Add filters and breakdowns
7. Save the visualization

---

## Dashboards

Dashboards combine multiple visualizations into a single view.

### Creating a Dashboard

1. Go to **Analytics** > **Dashboards**
2. Click **Create dashboard**
3. Click **Add panel**
4. Select existing visualizations or create new ones
5. Resize and arrange panels by dragging
6. Add filters that apply to all panels
7. Save the dashboard

### Dashboard Best Practices

- Add a **time filter** (global control)
- Use **controls** for interactive filtering (dropdowns, sliders)
- Group related panels together
- Add text panels as headers/descriptions
- Use consistent color schemes

---

## Dev Tools (Console)

Dev Tools lets you run Elasticsearch queries directly from Kibana.

**Access**: Menu > Management > Dev Tools

```
GET /logs-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "error" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "by_service": {
      "terms": { "field": "service" }
    }
  }
}
```

Dev Tools features:
- Auto-complete
- Request history
- JSON formatting
- Multiple request execution

---

## Alerting

Create alerts when data meets certain conditions.

### Create a Rule

1. Go to **Stack Management** > **Rules**
2. Click **Create rule**
3. Choose rule type (e.g., Elasticsearch query)
4. Define the condition (e.g., count > 10 errors in 5 min)
5. Set the action (email, Slack, PagerDuty, webhook)
6. Set the check interval

### Example Alert: Too Many Errors

```json
// Alert when error count > 50 in the last 5 minutes
{
  "index": "logs-*",
  "timeField": "@timestamp",
  "aggType": "count",
  "thresholdComparator": ">",
  "threshold": [50],
  "timeWindowSize": 5,
  "timeWindowUnit": "m",
  "filter": [{ "term": { "level": "error" } }]
}
```

---

## Kibana Configuration

Key settings in `kibana.yml`:

```yaml
# Server
server.port: 5601
server.host: "0.0.0.0"
server.name: "my-kibana"

# Elasticsearch
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "changeme"

# Logging
logging.appenders.default.type: "console"
logging.root.level: "info"

# Security
xpack.security.enabled: true
xpack.encryptedSavedObjects.encryptionKey: "a-32-character-long-key-here!!"
```

---

## Saved Objects

You can export and import Kibana saved objects (dashboards, visualizations, data views):

1. Go to **Stack Management** > **Saved Objects**
2. Select objects to export
3. Click **Export** (downloads `.ndjson` file)
4. To import: click **Import** and upload the file

This is useful for:
- Sharing dashboards between teams
- Backing up Kibana configuration
- Moving to a new environment

---

## Canvas (Custom Presentations)

Canvas creates pixel-perfect infographic-style reports using live Elasticsearch data.

- Supports custom layouts, images, colors
- Uses SQL-like expressions to query data
- Good for executive dashboards and reports

---

## Exercises

1. Set up a data view for `logs-*` in Kibana
2. Use **Discover** to find all `error` level logs from the past 24 hours
3. Use KQL to filter logs where `service` is `api` AND `status_code` >= 500
4. Create a **bar chart** showing log count grouped by `level`
5. Create a **line chart** showing error count over time (per hour)
6. Create a **data table** showing top 10 services by request count
7. Build a dashboard combining all three visualizations

---

## Summary

- **Discover**: Search and explore raw log data with KQL
- **Lens**: Drag-and-drop visualization builder
- **Dashboards**: Combine visualizations into unified views
- **Dev Tools**: Run Elasticsearch queries from the browser
- **Alerting**: Get notified when thresholds are crossed
- Use **Saved Objects** to export/import dashboards

---

**Previous:** [Day 4 - Logstash](../day-4/README.md) | **Next:** [Day 6 - Beats](../day-6/README.md)
