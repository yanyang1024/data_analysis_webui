# Data Analysis Report - Grader Agent

Evaluate test case execution results for the Data Analysis Report skill.

## Role

The Grader reviews execution transcripts and outputs from data analysis test cases, determining whether each expectation passes or fails with clear evidence.

## Inputs

- **expectations**: List of expectations from evals.json
- **transcript_path**: Path to execution transcript (markdown)
- **outputs_dir**: Directory containing output files (JSON responses, reports, etc.)

## Process

### Step 1: Read the Transcript

Read the complete execution transcript and note:
1. The user's analysis request
2. API calls made (base URL, endpoints used)
3. Command-line arguments passed
4. Error handling and responses
5. Final outputs generated

### Step 2: Examine Output Files

Check outputs_dir for:
1. **JSON response files** - Parse structure for required fields
2. **Generated reports** - Verify .docx files were created
3. **Transcript files** - Review execution steps
4. **Error logs** - Check for failures or workarounds

### Step 3: Evaluate Each Expectation

For each expectation:

**API-Related Expectations:**
- Check if the API endpoint was called correctly
- Verify request parameters (excel_path, user_prompt, etc.)
- Validate response structure (healthz, match, analyze fields)

**Script Behavior Expectations:**
- Verify command-line flags used correctly (--select-indicators, --quiet, etc.)
- Check exit codes via $? or error handling
- Confirm JSON parsing with jq or similar tools

**Output Quality Expectations:**
- Verify .docx file path exists in response
- Check indicator_names array is populated
- Validate time_window structure (type: "relative" or "absolute")
- Confirm required JSON fields are present

**Error Handling Expectations:**
- Look for try-except blocks in Python scripts
- Check for validation of inputs
- Verify graceful failure handling

### Step 4: Data Analysis Specific Checks

For this skill, pay special attention to:

1. **Excel File Handling**
   - File paths are absolute (not relative)
   - File extension is .xlsx
   - File exists before API call

2. **API Integration**
   - Base URL is correct (default: http://127.0.0.1:8001)
   - Endpoints called: /healthz, /analyze/match, /analyze
   - Request format: Form data for /analyze/match, JSON for /analyze

3. **Time Windows**
   - Relative times: "最近一年", "最近半年", "最近30天"
   - Absolute times: "2024-01-01 to 2024-12-31"
   - time_window field populated correctly

4. **Metric Matching**
   - indicator_names array from match endpoint
   - selected_indicator_names used for manual selection
   - Ambiguity handling when multiple similar columns exist

5. **Report Generation**
   - .docx file created at specified path
   - Report contains charts, statistics, insights
   - File is valid Word document format

### Step 5: Extract and Verify Claims

Look for implicit claims in outputs:
- "Analysis completed successfully" - verify exit code 0
- "All metrics analyzed" - check indicator_names count
- "Report saved to..." - verify file exists
- "Time range: X to Y" - validate time_window fields

### Step 6: Write Grading Results

Save to `{outputs_dir}/../grading.json` with the standard schema.

## Grading Criteria

**PASS when:**
- Transcript shows the specific command/API call
- Output JSON contains the required field with correct value
- File exists and has expected properties
- Error handling is demonstrated in code/script

**FAIL when:**
- No evidence of the expected action
- Output missing required fields
- Wrong API endpoint or parameters used
- File doesn't exist or is wrong format
- Exit code indicates failure (non-zero)

## Output Format

```json
{
  "expectations": [
    {
      "text": "The script calls the data analysis API with the Excel file path",
      "passed": true,
      "evidence": "Transcript shows: python call_data_analysis_api.py --excel-path /data/sales.xlsx --user-prompt 'analyze trends'"
    },
    {
      "text": "The output includes a .docx file path in the response JSON",
      "passed": true,
      "evidence": "Response JSON contains: 'report_path': '/home/data/reports/report_20250301.docx'"
    },
    {
      "text": "The response JSON contains indicator_names field",
      "passed": false,
      "evidence": "Response JSON missing 'indicator_names' field, only has 'status' and 'message'"
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  },
  "execution_metrics": {
    "tool_calls": {"Read": 3, "Write": 1, "Bash": 5},
    "total_tool_calls": 9,
    "total_steps": 4,
    "errors_encountered": 0,
    "output_chars": 2500,
    "transcript_chars": 1800
  },
  "timing": {
    "executor_duration_seconds": 45.0,
    "grader_duration_seconds": 8.0,
    "total_duration_seconds": 53.0
  },
  "claims": [
    {
      "claim": "Analysis completed for Q4 2024 data",
      "type": "factual",
      "verified": true,
      "evidence": "time_window shows 'Q4 2024' in response JSON"
    }
  ],
  "user_notes_summary": {
    "uncertainties": [],
    "needs_review": [],
    "workarounds": []
  },
  "eval_feedback": {
    "suggestions": [
      {
        "assertion": "The script calls the data analysis API",
        "reason": "Too generic - should specify which endpoint (/analyze/match vs /analyze) and verify request format"
      }
    ],
    "overall": "Consider adding assertions to verify request/response formats match API spec"
  }
}
```

## Data Analysis Skill Specific Notes

1. **API Endpoints**: The skill has two main endpoints:
   - `/analyze/match` - For metric disambiguation (uses Form data)
   - `/analyze` - For actual analysis (uses JSON body)
   Verify the correct endpoint is used for each phase.

2. **JSON Schema**: The response must follow this structure:
   ```json
   {
     "healthz": {"status": "ok"},
     "match": {"status": "ok", "indicator_names": ["metric1", "metric2"]},
     "analyze": {
       "report_path": "/path/to/report.docx",
       "time_window": {"type": "relative", "value": "最近一年"},
       "indicator_names": ["metric1"],
       "sheet_name": "Sheet1",
       "date_column": "日期"
     }
   }
   ```

3. **Exit Codes**:
   - 0: Success
   - 2: HTTP error
   - 3: Request/connection error
   - 4: Validation error
   - 5: Unexpected error

4. **Common Issues to Check**:
   - Relative paths instead of absolute paths for Excel files
   - Missing --base-url when API is not on localhost:8001
   - Incorrect time window format (should match LLM expectations)
   - Timeout issues for large files (>10K rows)
   - Metric ambiguity not handled properly
