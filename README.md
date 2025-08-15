```json
{
  "version": "1.0",
  "baseUrl": "https://api.example.com",
  "globals": {
    "headers": {
      "Authorization": "Bearer {{token}}"
    },
    "vars": {
      "tenant": "acme"
    },
    "retry": {
      "maxAttempts": 3,
      "backoffMs": 200,
      "jitter": true,
      "retryOn": [429, 503]
    },
    "timeouts": {
      "connectMs": 3000,
      "readMs": 5000
    }
  },
  "requests": [
    {
      "name": "login",
      "method": "POST",
      "path": "/auth/login",
      "body": {
        "user": "{{user}}",
        "pass": "{{pass}}"
      },
      "save": {
        "token": "$.token"
      },
      "assert": {
        "status": 200,
        "jsonPath": {
          "$.token": "present"
        }
      }
    },
    {
      "name": "getUsers",
      "method": "GET",
      "path": "/tenants/{{tenant}}/users",
      "query": {
        "limit": "25"
      },
      "assert": {
        "status": 200,
        "jsonPath": {
          "$.items.length()": ">=1",
          "$.items[0].id": "present"
        }
      }
    }
  ],
  "flow": {
    "variables": {
      "user": "user-{{vu}}",
      "pass": "demo"
    },
    "thinkTimeMs": {
      "min": 50,
      "max": 150
    },
    "onError": "continue"
  },
  "load": {
    "model": "closed",
    "users": 50,
    "rampUp": "30s",
    "holdFor": "5m",
    "iterations": 3,
    "warmup": "10s",
    "stopOn": {
      "errorRatePct": 5,
      "p95LtMs": 800
    }
  }
}
```
