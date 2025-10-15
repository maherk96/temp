```json
[
  {
    "type": "MOST_TEST_CASES_FAILED",
    "displayName": "Most Test Cases Failed",
    "graphType": "BAR",
    "widgetConfig": {
      "type": "MOST_TEST_CASES_FAILED",
      "fields": {
        "appName": {
          "dataType": "string",
          "description": "The name of the application to filter test results for"
        },
        "days": {
          "dataType": "integer",
          "description": "The number of days of test history to include"
        },
        "includeRegression": {
          "dataType": "boolean",
          "description": "Whether to include regression test results"
        }
      },
      "example": {
        "appName": "Fusion Algo",
        "days": 14,
        "includeRegression": true
      }
    }
  }
]

```
