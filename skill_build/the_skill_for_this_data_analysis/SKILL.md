---
name: data-analysis-report
description: Generate comprehensive data analysis reports from Excel files using LLM-powered insights. ALWAYS use this skill whenever the user wants to analyze Excel data, generate reports from spreadsheets, examine data trends, compare business metrics, create statistical analyses, visualize time-series data, generate business intelligence reports, perform data exploration on .xlsx files, summarize spreadsheet data, correlate multiple metrics, or needs to convert Excel data into professional Word documents with charts. This skill handles natural language analysis requests, works offline with Ollama/vLLM, and outputs formatted Word reports with AI-generated insights, statistical summaries, and visualizations. Even if the user doesn't explicitly mention "report" or "analysis", if they're working with Excel data and want insights, trends, comparisons, or visualizations, use this skill.
metadata:
  author: data-analysis-webui
  version: "3.0.0"
  license: MIT
---

# Data Analysis Report Generator

Automatically analyze Excel data and generate comprehensive Word reports with charts, statistics, and AI-powered insights. Works offline with Ollama or vLLM models.

## How It Works

1. **Excel Parsing** - Automatically detects date columns and numeric metrics
2. **Intent Understanding** - LLM parses your natural language request to extract metrics and time windows
3. **Indicator Disambiguation** - Handles multiple similar column names by asking for clarification
4. **Statistical Analysis** - Computes trends, percentiles, outliers, and correlations
5. **Report Generation** - Creates Word document with charts, tables, and AI-written insights

## Usage

```bash
python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
  --base-url http://127.0.0.1:8001 \
  --excel-path /path/to/data.xlsx \
  --user-prompt "your analysis request here"
```

**Required Arguments:**
- `--base-url` - API server URL (default: `http://127.0.0.1:8001`)
- `--excel-path` - Absolute path to Excel file (.xlsx)
- `--user-prompt` - Natural language analysis request

**Optional Arguments:**
- `--sheet-name` - Specific sheet name (auto-detected if omitted)
- `--use-llm-structure` / `--no-use-llm-structure` - Use LLM for Excel structure detection (default: enabled)
- `--select-indicators` - Manually specify metrics (skips auto-disambiguation)
- `--timeout` - Request timeout in seconds (default: 120)

## Examples

### Basic Trend Analysis

```bash
python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
  --base-url http://127.0.0.1:8001 \
  --excel-path /home/data/sales_2024.xlsx \
  --user-prompt "分析最近一个季度的销量趋势"
```

### Multi-Metric Comparison

```bash
python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
  --base-url http://127.0.0.1:8001 \
  --excel-path /home/data/production.xlsx \
  --user-prompt "分析最近半年产量和销量的相关性，并给出建议"
```

### Specific Time Range

```bash
python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
  --base-url http://127.0.0.1:8001 \
  --excel-path /home/data/financial.xlsx \
  --user-prompt "分析 2024-01-01 至 2024-06-30 期间的收入与支出"
```

### Manual Metric Selection

```bash
python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
  --base-url http://127.0.0.1:8001 \
  --excel-path /home/data/warehouse.xlsx \
  --user-prompt "分析库存周转情况" \
  --select-indicators "库存数量" "入库数量" "出库数量"
```

### With Custom Sheet and Timeout

```bash
python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
  --base-url http://127.0.0.1:8001 \
  --excel-path /home/data/report.xlsx \
  --sheet-name "Q4 2024" \
  --user-prompt "分析本季度所有指标的表现" \
  --timeout 300
```

## Output

### Console Output

```bash
Preparing analysis...
✓ Health check passed
✓ Metrics matched: 产量, 销量
✓ Time window resolved: 最近一年
✓ Analysis complete

Report saved to: /home/data/reports/report_20250301_143025.docx

Summary:
- Metrics: 产量, 销量
- Time window: 最近一年 (2024-03-01 to 2025-03-01)
- Records: 365
- Sheet: Sheet1
```

### JSON Output (stdout)

```json
{
  "healthz": {"status": "ok"},
  "match": {
    "status": "ok",
    "indicator_names": ["产量", "销量"]
  },
  "selected_indicator_names": ["产量", "销量"],
  "analyze": {
    "report_path": "/home/data/reports/report_20250301_143025.docx",
    "time_window": {"type": "relative", "value": "最近一年"},
    "indicator_names": ["产量", "销量"],
    "sheet_name": "Sheet1",
    "date_column": "日期"
  }
}
```

### Generated Report Contents

The Word document (`.docx`) includes:
- **Time-series charts** for each metric with trend lines
- **Statistical summary table**: mean, min, max, median, std, P5, P95, CV, outliers
- **Period-over-period comparison**: start vs end values, absolute & % change
- **AI-generated insights**: trend analysis, anomaly detection, recommendations
- **Drift analysis**: linear regression slope for trend detection
- **Professional formatting**: Times New Roman (Latin), 宋体 (CJK)

## Excel File Requirements

**Required:**
- ✅ Must have a date column (auto-detected)
- ✅ Must have numeric metric columns
- ✅ First row must be column headers

**Supported:**
- ✅ Multiple sheets (auto-selects largest)
- ✅ Various date formats (Excel serial, ISO, etc.)
- ✅ Missing values (handled gracefully)
- ✅ Large files (100K+ rows with `use_llm_structure=false`)

**Not Supported:**
- ❌ Merged cells
- ❌ Pivot tables
- ❌ Protected sheets

## Time Window Formats

Supports both relative and absolute time ranges:

| Relative | Absolute |
|----------|----------|
| "最近一年" | "2024-01-01 至 2024-12-31" |
| "最近半年" | "2024 Q1 to 2024 Q2" |
| "最近30天" | "January 2024" |
| "本季度" | "last week" |

## Metric Disambiguation Flow

When multiple similar column names exist (e.g., "产量", "半钢胎产量", "全钢胎产量"):

```
User: "分析产量趋势"
↓
System detects ambiguity → Returns candidates
↓
User selects: "产量", "半钢胎产量"
↓
Analysis proceeds with selected metrics
```

**To skip disambiguation**, use `--select-indicators` to specify exact column names.

## Present Results to User

Always present in this format:

```
✓ Analysis complete!

📊 Report: /path/to/report_20250301_143025.docx
📈 Metrics: 产量, 销量
📅 Period: 最近一年 (2024-03-01 to 2025-03-01)
📁 Source: Sheet1

Key findings:
- 产量平均值为 1050，较期初增长 15.3%
- 销量呈现上升趋势，斜率为 2.3/天
- 两指标相关性: 0.87 (强正相关)

打开 Word 文档查看完整图表与详细分析。
```

## API Server Setup

The skill requires a running Data Analysis API server:

### Quick Start

```bash
# 1. Install dependencies
cd /path/to/data_analysis_webui
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 2. Configure Ollama/vLLM (edit api/config.azure.json)
# Already configured for: http://172.24.16.1:11434 with qwen3:14b

# 3. Start server
uvicorn src.main:app --host 0.0.0.0 --port 8001

# 4. Verify
curl http://127.0.0.1:8001/healthz
```

### Background Service

```bash
# Start in background
nohup uvicorn src.main:app --host 0.0.0.0 --port 8001 > api.log 2>&1 &

# Check logs
tail -f api.log

# Stop service
pkill -f "uvicorn src.main:app"
```

## Troubleshooting

### Service Not Running

```
❌ Error: Connection refused

Solution: Start the API server
  uvicorn src.main:app --host 0.0.0.0 --port 8001
```

### Ollama Connection Failed

```
❌ Error: LLM request timeout

Solution: Check Ollama service
  curl http://172.24.16.1:11434/v1/models

If failing, ensure Ollama is running:
  ollama serve  # on Ollama machine
```

### Model Not Found

```
❌ Error: model 'qwen3:14b' not found

Solution: Pull the model on Ollama machine
  ollama pull qwen3:14b
```

### Excel Parse Error

```
❌ Error: 未匹配到有效指标列

Solution: Use --select-indicators to specify exact column names
  --select-indicators "产量" "销量"
```

### Timeout

```
❌ Error: Timeout after 120s

Solution: Increase timeout for large files
  --timeout 600
```

### Common Errors Quick Reference

| Error | Cause | Fix |
|-------|-------|-----|
| `healthz` failed | API not started | `uvicorn src.main:app --host 0.0.0.0 --port 8001` |
| `match` returns `not_found` | Prompt too vague | Be specific about metric names |
| `analyze` returns 400 | File not found | Use absolute path, check file exists |
| `analyze` returns 400 | No data in time window | Check date range in Excel |
| LLM timeout | Model too slow | Increase `API_TIMEOUT_MS` in config |

## Advanced Configuration

### Environment Variables

```bash
# Override config file location
export DATA_ANALYSIS_CONFIG_PATH=/custom/config.json

# Override report output directory
export DATA_ANALYSIS_OUTPUT_DIR=/custom/output

# Override API timeout
# Edit api/config.azure.json: "API_TIMEOUT_MS": 1200000
```

### Model Selection

Edit `api/config.azure.json`:

```json
{
  "Providers": [{
    "name": "ollama",
    "api_base_url": "http://172.24.16.1:11434/v1",
    "models": ["qwen3:14b", "qwen2.5:32b"],  // Available models
    "context_window_chars": 128000
  }],
  "Router": { "default": "ollama,qwen3:14b" }  // Default model
}
```

### Context Window Tuning

For large Excel files (>10K rows):

```json
{
  "MODEL_CONTEXT_WINDOW_CHARS": 256000,  // Increase context
  "RAW_FILE_CONTEXT_RATIO": 0.5  // Allow more raw data
}
```

## Best Practices

### Writing Good Prompts

✅ **Good prompts:**
- "分析最近一年产量和销量的趋势，比较两者的相关性"
- "分析2024-01至2024-06库存周转情况，找出异常点"
- "对比Q1和Q2的各指标表现，给出改进建议"

❌ **Poor prompts:**
- "分析一下" (too vague)
- "最近怎么样" (no time range)
- "帮我看看" (no metrics)

### For Large Files

```bash
# Disable LLM structure detection (faster, rule-based)
--no-use-llm-structure

# Increase timeout
--timeout 600
```

### For Batch Processing

```bash
# Process multiple files
for file in data/*.xlsx; do
  python scripts/call_data_analysis_api.py \
    --base-url http://127.0.0.1:8001 \
    --excel-path "$file" \
    --user-prompt "分析所有指标的月度趋势"
done
```

## Integration Examples

### Python Script Integration

```python
import subprocess
import json

def analyze_excel(excel_path, prompt):
    result = subprocess.run([
        "python", "/mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py",
        "--base-url", "http://127.0.0.1:8001",
        "--excel-path", excel_path,
        "--user-prompt", prompt
    ], capture_output=True, text=True)

    return json.loads(result.stdout)

# Usage
report_info = analyze_excel("/data/sales.xlsx", "分析最近一季度的销量趋势")
print(f"Report: {report_info['analyze']['report_path']}")
```

### Shell Script Integration

```bash
#!/bin/bash
analyze() {
  local file=$1
  local prompt=$2
  python /mnt/skills/user/data-analysis-report/scripts/call_data_analysis_api.py \
    --base-url http://127.0.0.1:8001 \
    --excel-path "$file" \
    --user-prompt "$prompt" | jq -r '.analyze.report_path'
}

# Usage
report_path=$(analyze "/data/sales.xlsx" "分析销量")
echo "Report saved to: $report_path"
```

## WebUI Alternative

For interactive multi-round analysis, use the Gradio WebUI:

```bash
python src/gradio_app.py
# Access at http://localhost:5600
```

WebUI features:
- File upload via browser
- Multi-turn conversation
- Report preview and download
- Comprehensive summary generation
- Iterative summary refinement

## See Also

- **Deployment Guide**: `DEPLOYMENT_GUIDE.md` - Complete server setup guide
- **Quick Start**: `QUICKSTART.md` - 5-minute getting started
- **API Documentation**: http://127.0.0.1:8001/docs (when server is running)
- **Project README**: `README.md` - Project overview
