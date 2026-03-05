# Data Analysis API Reference

Complete API specification for the Data Analysis WebUI service.

## Base URL

```
http://127.0.0.1:8001
```

Configurable via `--base-url` argument or environment variable.

## Endpoints

### GET /healthz

Health check endpoint.

**Request:**
```bash
curl http://127.0.0.1:8001/healthz
```

**Response (200 OK):**
```json
{
  "status": "ok",
  "service": "data-analysis-api",
  "version": "1.0.0"
}
```

**Response (503 Service Unavailable):**
```json
{
  "status": "error",
  "message": "Database connection failed"
}
```

---

### POST /analyze/match

Metric disambiguation endpoint. Returns candidate metrics matching the user's prompt.

**Request Format:** Form data (NOT JSON)

```bash
curl -X POST "http://127.0.0.1:8001/analyze/match" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "excel_path=/path/to/data.xlsx" \
  -d "user_prompt=分析产量趋势" \
  -d "sheet_name=Sheet1" \
  -d "use_llm_structure=true"
```

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| excel_path | string | Yes | Absolute path to Excel file (.xlsx) |
| user_prompt | string | Yes | Natural language analysis request |
| sheet_name | string | No | Specific sheet name (auto-detect if omitted) |
| use_llm_structure | boolean | No | Use LLM for Excel parsing (default: true) |

**Response (200 OK) - Unambiguous Match:**
```json
{
  "status": "ok",
  "indicator_names": ["产量", "销量"],
  "sheet_name": "Sheet1",
  "date_column": "日期"
}
```

**Response (200 OK) - Ambiguous Match:**
```json
{
  "status": "ambiguous",
  "candidates": [
    {
      "name": "产量",
      "display": "产量",
      "similarity": 0.95
    },
    {
      "name": "半钢胎产量",
      "display": "半钢胎产量",
      "similarity": 0.88
    },
    {
      "name": "全钢胎产量",
      "display": "全钢胎产量",
      "similarity": 0.87
    }
  ],
  "sheet_name": "Sheet1"
}
```

**Response (404 Not Found) - No Metrics:**
```json
{
  "status": "not_found",
  "message": "未匹配到有效指标列",
  "suggestions": [
    "Check if Excel file has numeric columns",
    "Verify column headers are in first row",
    "Try using --select-indicators to specify exact column names"
  ]
}
```

**Response (400 Bad Request):**
```json
{
  "detail": "Excel file not found: /path/to/data.xlsx"
}
```

---

### POST /analyze

Execute analysis and generate report.

**Request Format:** JSON body

```bash
curl -X POST "http://127.0.0.1:8001/analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "excel_path": "/path/to/data.xlsx",
    "user_prompt": "分析最近一年产量趋势",
    "sheet_name": "Sheet1",
    "use_llm_structure": true,
    "selected_indicator_names": ["产量", "销量"]
  }'
```

**Request Body:**
```json
{
  "excel_path": "/path/to/data.xlsx",
  "user_prompt": "分析最近一年产量趋势",
  "sheet_name": "Sheet1",
  "use_llm_structure": true,
  "selected_indicator_names": ["产量", "销量"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| excel_path | string | Yes | Absolute path to Excel file |
| user_prompt | string | Yes | Natural language analysis request |
| sheet_name | string | No | Sheet name (auto-detect if omitted) |
| use_llm_structure | boolean | No | Use LLM for parsing (default: true) |
| selected_indicator_names | array[string] | No | Manual metric selection (skips disambiguation) |

**Response (200 OK):**
```json
{
  "report_path": "/home/data/reports/report_20250301_143025.docx",
  "time_window": {
    "type": "relative",
    "value": "最近一年",
    "start_date": "2024-03-01",
    "end_date": "2025-03-01"
  },
  "indicator_names": ["产量", "销量"],
  "sheet_name": "Sheet1",
  "date_column": "日期",
  "statistics": {
    "产量": {
      "count": 365,
      "mean": 1050.5,
      "min": 850,
      "max": 1250,
      "median": 1048,
      "std": 120.3,
      "p5": 870,
      "p95": 1220
    },
    "销量": {
      "count": 365,
      "mean": 980.2,
      "min": 750,
      "max": 1150,
      "median": 975,
      "std": 110.8,
      "p5": 790,
      "p95": 1120
    }
  },
  "correlation": {
    "产量_vs_销量": 0.87
  }
}
```

**Response (400 Bad Request) - File Not Found:**
```json
{
  "detail": "Excel file not found: /path/to/data.xlsx"
}
```

**Response (400 Bad Request) - No Data in Time Window:**
```json
{
  "detail": "No data found in the specified time range: 2025-01-01 to 2025-12-31"
}
```

**Response (400 Bad Request) - No Metrics Matched:**
```json
{
  "detail": "未匹配到有效指标列"
}
```

**Response (500 Internal Server Error):**
```json
{
  "detail": "LLM request timeout",
  "error": "Connection to Ollama failed after 120s"
}
```

---

## Time Window Formats

### Relative Time Windows

| Format | Description | Example Resolution |
|--------|-------------|-------------------|
| "最近一年" | Last year | current_date - 365 days |
| "最近半年" | Last 6 months | current_date - 180 days |
| "最近30天" | Last 30 days | current_date - 30 days |
| "本季度" | Current quarter | Current quarter start to now |
| "上个月" | Last month | Previous month full range |

### Absolute Time Windows

| Format | Description |
|--------|-------------|
| "2024-01-01 至 2024-12-31" | ISO date range with Chinese separator |
| "2024-01-01 to 2024-12-31" | ISO date range with English separator |
| "January 2024" | Full month |
| "Q1 2024" | Quarter |

---

## Excel File Requirements

### Required Structure

```
┌─────────────────────────────────────┐
│ Date       │ Metric1 │ Metric2 │ ... │  ← First row: headers
├─────────────────────────────────────┤
│ 2024-01-01 │ 100     │ 200     │ ... │
│ 2024-01-02 │ 105     │ 205     │ ... │
│ ...        │ ...     │ ...     │ ... │
└─────────────────────────────────────┘
```

**Requirements:**
- ✅ First row must contain column headers
- ✅ At least one date column (auto-detected)
- ✅ At least one numeric metric column
- ✅ Dates in Excel serial format or ISO format

**Supported:**
- ✅ Multiple sheets (auto-selects largest)
- ✅ Missing values (handled gracefully)
- ✅ Large files (100K+ rows with `use_llm_structure=false`)

**Not Supported:**
- ❌ Merged cells in data range
- ❌ Pivot tables as data source
- ❌ Password-protected sheets

---

## Error Codes

| Exit Code | Description |
|-----------|-------------|
| 0 | Success |
| 1 | Reserved |
| 2 | HTTP error (4xx, 5xx responses) |
| 3 | Request/connection error (timeout, connection refused) |
| 4 | Validation error (invalid input) |
| 5 | Unexpected error (unhandled exception) |

---

## Common Request Patterns

### Pattern 1: Simple Trend Analysis

```bash
# Step 1: Match metrics
curl -X POST "http://127.0.0.1:8001/analyze/match" \
  -d "excel_path=/data/sales.xlsx" \
  -d "user_prompt=分析销量趋势"

# Step 2: If ambiguous, show user candidates
# Step 3: Run analysis with selected metrics
curl -X POST "http://127.0.0.1:8001/analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "excel_path": "/data/sales.xlsx",
    "user_prompt": "分析销量趋势",
    "selected_indicator_names": ["销量"]
  }'
```

### Pattern 2: Multi-Metric Analysis

```bash
# Skip disambiguation by specifying metrics directly
curl -X POST "http://127.0.0.1:8001/analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "excel_path": "/data/production.xlsx",
    "user_prompt": "分析产量和销量的相关性",
    "selected_indicator_names": ["产量", "销量"]
  }'
```

### Pattern 3: Large File Processing

```bash
# Disable LLM structure detection for faster processing
curl -X POST "http://127.0.0.1:8001/analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "excel_path": "/data/large_file.xlsx",
    "user_prompt": "分析所有指标",
    "use_llm_structure": false,
    "selected_indicator_names": ["metric1", "metric2", "metric3"]
  }'
```

---

## Rate Limiting and Timeouts

- **Default timeout**: 120 seconds (configurable via `--timeout`)
- **Large file timeout**: 600 seconds recommended for files >10K rows
- **No rate limiting**: Built for offline enterprise use

---

## Response Headers

```
Content-Type: application/json
Access-Control-Allow-Origin: *
Server: uvicorn
```

---

## Version Compatibility

- **API Version**: 1.0.0
- **Minimum Python**: 3.9
- **Compatible Models**: Ollama (qwen2.5:14b, qwen3:14b), vLLM (any OpenAI-compatible)

---

## Testing the API

### Quick Health Check

```bash
curl http://127.0.0.1:8001/healthz
```

### Full Analysis Test

```python
import requests

# Test file setup
excel_path = "/path/to/test_data.xlsx"
base_url = "http://127.0.0.1:8001"

# Match metrics
match_response = requests.post(
  f"{base_url}/analyze/match",
  data={
    "excel_path": excel_path,
    "user_prompt": "分析产量和销量",
    "use_llm_structure": True
  }
)

# Check response
if match_response.status_code == 200:
    match_data = match_response.json()

    if match_data["status"] == "ok":
        indicators = match_data["indicator_names"]
    elif match_data["status"] == "ambiguous":
        # User selects from candidates
        indicators = [match_data["candidates"][0]["name"]]

    # Run analysis
    analyze_response = requests.post(
      f"{base_url}/analyze",
      json={
        "excel_path": excel_path,
        "user_prompt": "分析产量和销量",
        "selected_indicator_names": indicators
      },
      timeout=600
    )

    if analyze_response.status_code == 200:
        result = analyze_response.json()
        print(f"Report: {result['report_path']}")
```

---

## Troubleshooting

### Connection Refused

**Error:** `requests.exceptions.ConnectionError: Connection refused`

**Solutions:**
1. Start API server: `uvicorn src.main:app --host 0.0.0.0 --port 8001`
2. Check firewall settings
3. Verify port 8001 is not in use

### LLM Timeout

**Error:** `LLM request timeout after 120s`

**Solutions:**
1. Check Ollama service: `curl http://172.24.16.1:11434/v1/models`
2. Increase timeout in `api/config.azure.json`: `"API_TIMEOUT_MS": 300000`
3. Use smaller model or faster hardware

### File Not Found

**Error:** `400 Bad Request: Excel file not found`

**Solutions:**
1. Use absolute path: `/home/user/data.xlsx` not `data.xlsx`
2. Verify file exists: `ls -la /path/to/file.xlsx`
3. Check file permissions: `chmod +r /path/to/file.xlsx`

### No Metrics Matched

**Error:** `未匹配到有效指标列`

**Solutions:**
1. Check Excel has numeric columns
2. Verify headers are in first row
3. Use `--select-indicators` to specify exact column names
4. Check if LLM is running (for `use_llm_structure=true`)

---

## See Also

- **Main Documentation**: `SKILL.md`
- **Usage Examples**: `EXAMPLES.md`
- **Deployment Guide**: `DEPLOYMENT_GUIDE.md` (project root)
- **API Interactive Docs**: http://127.0.0.1:8001/docs (when server running)
