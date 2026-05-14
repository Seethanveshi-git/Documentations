# Automate the Key Request Using the Settings API (Dynatrace)

This guide explains how to automate **Key Request creation** using  Settings API.

---

## Prerequisites

To make the request, you need:

- Environment URL  
- API Token  
- Service ID  
- Endpoints  

These are required to mark a request as a **Key Request**.

---

## Token Permissions

Ensure your API token has the following permissions:

- Read configuration  
- Write configurations  
- `settings.read`  
- `settings.write`  
- DataExport  

---

## Step 1: Verify API Access

Before making API calls, verify:

- API token is valid  
- API is reachable  
- Permissions are correct  

## Check API connectivity

## Curl API

```bash
curl -X GET "https://<env-Id>.live.dynatrace.com/api/v1/config/clusterversion" \
-H "Authorization: Api-Token YOUR_API_TOKEN"
```

## NodeJs code

```bash
const axios = require("axios");

const envId = "YOUR_ENV_ID";
const apiToken = "YOUR_API_TOKEN";

const url = `https://${envId}.live.dynatrace.com/api/v1/config/clusterversion`;

async function getClusterVersion() {
  try {
    const response = await axios.get(url, {
      headers: {
        "Authorization": `Api-Token ${apiToken}`,
        "Accept": "application/json",
      },
    });

    console.log(response.data);
  } catch (error) {
    console.error("Error fetching cluster version:", error.message);
  }
}

getClusterVersion();
```

## Expected Response

```json
{
  "version": "1.xxx.xxx"
}
```



## Step 2: Find Service IDs

Use this API to fetch all service IDs:

## Curl API

```bash
curl -X GET "https://<env-Id>.live.dynatrace.com/api/v2/entities?entitySelector=type(SERVICE)" \
-H "Authorization: Api-Token <Token>"
```

## NodeJs code

```bash
const axios = require("axios");

const envId = "YOUR_ENV_ID";
const apiToken = "YOUR_API_TOKEN";

const url = `https://${envId}.live.dynatrace.com/api/v2/entities`;

async function getServices() {
  try {
    const response = await axios.get(url, {
      headers: {
        "Authorization": `Api-Token ${apiToken}`,
        "Accept": "application/json",
      },
      params: {
        entitySelector: "type(SERVICE)",
      },
    });

    console.log(response.data);
  } catch (error) {
    if (error.response) {
      console.error("API Error:", error.response.status, error.response.data);
    } else {
      console.error("Request Error:", error.message);
    }
  }
}

getServices();
```

## Expected Response

```json
{
  "entities": [
    {
      "entityId": "SERVICE-123456789",
      "type" : "SERVICE"
      "displayName": "auth-service"
    },
    {
      "entityId": "SERVICE-987654321",
      "type" : "SERVICE"
      "displayName": "expense-service"
    }
  ]
}
```

## Step 3: Implementation of Marks as a Key Request (Multiple Services & Endpoints)

## NodeJs Code




```bash
const axios = require("axios");

const ENV_URL = "<Environment-URL>";
const API_TOKEN = "<YOUR_TOKEN>";

const services = [
  {
    serviceId: "<SERVICE-ID-1>",
    endpoints: ["/api/auth/login", "/api/auth/signup"]
  },
  {
    serviceId: "<SERVICE-ID-2>",
    endpoints: ["/api/groups", "/api/expense"]
  },
  {
    serviceId: "<SERVICE-ID-3>",
    endpoints: ["/api/payment", "/api/orders"]
  }
];

async function createKeyRequests() {
  try {
    const payload = services.flatMap((service) =>
      service.endpoints.map((ep) => ({
        schemaId: "builtin:settings.subscriptions.service",
        scope: service.serviceId,
        value: {
          enabled: true,
          requestName: ep,
          conditions: [
            {
              attribute: "REQUEST_NAME",
              comparisonInfo: {
                type: "STRING",
                operator: "EQUALS",
                value: ep
              }
            }
          ]
        }
      }))
    );

    const response = await axios.post(
      `${ENV_URL}/api/v2/settings/objects`,
      payload,
      {
        headers: {
          Authorization: `Api-Token ${API_TOKEN}`,
          "Content-Type": "application/json"
        }
      }
    );

    console.log("SUCCESS: Key Requests created");
    console.log(JSON.stringify(response.data, null, 2));

  } catch (error) {
    console.error("ERROR creating Key Requests:");
    console.error(
      JSON.stringify(
        error.response?.data || error.message,
        null,
        2
      )
    );
  }
}

createKeyRequests();
```

## Expected Response

```josn
[
  {
    "objectId": "<ObjectId>",
    "updateToken": "<Update-Token>",
    "inserted": true
  }
]
```


## Step 4: Verify Created Objects

```bash
curl -X GET \
"https://<env-id>.live.dynatrace.com/api/v2/settings/objects?schemaIds=builtin:settings.subscriptions.service" -H "Authorization: Api-Token <token>"
```
## Expected Response

```json
{
  "items": [
    {
      "objectId": "vu9U3hXa3q0fczW2",
      "updateToken": "abcdef1234567890",
      "scope": "SERVICE-1234567890ABCDE",
      "schemaId": "builtin:service.key-request",
      "value": {
        "requestName": "/api/auth/login",
        "enabled": true
      }
    }
  ]
}
```
