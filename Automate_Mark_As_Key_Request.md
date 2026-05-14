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

const ENV_URL = "<ENV_URL>";
const API_TOKEN = "<TOKEN>";

const services = [
    {
        serviceName: "frontend",
        endpoints: ["/api/auth/login", "/api/auth/signup"]
    },
    {
        serviceName: "ExpenseController",
        endpoints: ["/api/expense", "/api/expense/1"]
    },
    {
        serviceName: "GroupController",
        endpoints: ["/api/group", "/api/group/1"]
    }
];

async function getServiceIdByName(serviceName) {
    try {
        const response = await axios.get(`${ENV_URL}/api/v2/entities`, {
            params: {
                entitySelector: `type("SERVICE"),entityName("${serviceName}")`,
                fields: "firstSeenTms,lastSeenTms,properties",
            },
            headers: {
                Authorization: `Api-Token ${API_TOKEN}`
            }
        });

        const entities = response.data.entities || [];

        if (entities.length === 0) {
            throw new Error(`Service not found: ${serviceName}`);
        }

        entities.sort((a, b) => {
            if (b.lastSeenTms !== a.lastSeenTms) {
                return b.lastSeenTms - a.lastSeenTms;
            }
            
            return b.firstSeenTms - a.firstSeenTms;
        });

        const bestMatch = entities[0];

        return bestMatch.entityId;

    } catch (error) {
        throw new Error(
            error.response?.data
                ? JSON.stringify(error.response.data)
                : error.message
        );
    }
}

async function createKeyRequests() {
    try {
        const payload = [];

        for (const service of services) {
            // Wait for the ID to be found and returned
            const serviceId = await getServiceIdByName(service.serviceName);
            
            console.log(`Success: Using ID ${serviceId} for ${service.serviceName}`);

            payload.push({
                schemaId: "builtin:settings.subscriptions.service",
                scope: serviceId,
                value: {
                    keyRequestNames: service.endpoints
                }
            });
        }

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

        console.log("\nFINAL SUCCESS: Key Requests created in Dynatrace");
        console.log(JSON.stringify(response.data, null, 2));
    } catch (error) {
        console.error(
            "\nFINAL ERROR:",
            JSON.stringify(error.response?.data || error.message, null, 2)
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


## Special Case: Handling Multiple Service IDs (The “Ghost ID” Problem)

In real production environments, the Dynatrace API may occasionally return multiple entity IDs for the same service name.

---

## Why does this happen?

## Redeployments & Rolling Updates

Dynatrace may temporarily retain older service entities as inactive while simultaneously monitoring new active entities for the updated version.

---

## Infrastructure Changes

Migrations (like moving between namespaces), cluster changes, or service re-identifications can trigger the creation of a new Entity ID for the same logical service.

---

## API Time Window

By default, the API evaluates a recent time window (2 hours).

If a service was restarted, both the old and new IDs will appear in the search results.

---

## How Our Script Handles This

To prevent automation failures or targeting "stale" entities, the script applies a **Dual-Criteria Heuristic** based on both **Activity** and **Creation Time**.

---

## Step 1: Request Lifecycle Timestamps

We request specific fields from the API to understand the "Pulse" and "Birth" of each entity.

- `lastSeenTms`: The "Heartbeat" (Last time data was received)
- `firstSeenTms`: The "Birth Date" (When the ID was first created)

### API Request

```bash
const response = await axios.get(`${ENV_URL}/api/v2/entities`, {
    params: {
        entitySelector: `type("SERVICE"),entityName("${serviceName}")`,
        fields: "firstSeenTms,lastSeenTms",
    },
    headers: {
        Authorization: `Api-Token ${API_TOKEN}`
    }
});
```

---

## Step 2: Prioritize Activity (Double-Sort Logic)

The script sorts the results using two levels of priority:

## Primary Priority (`lastSeenTms`)

We prioritize the service that is currently communicating with Dynatrace.

---

## Secondary Priority (`firstSeenTms`)

If two services are active simultaneously (common during rolling deployments), we select the most recently created entity.

---

## Sorting Implementation

```bash
entities.sort((a, b) => {

    // 1. Check Heartbeat:
    // Pick the entity seen most recently
    if (b.lastSeenTms !== a.lastSeenTms) {
        return b.lastSeenTms - a.lastSeenTms;
    }

    // 2. Tie-breaker:
    // If both are active, pick the newest deployment
    return b.firstSeenTms - a.firstSeenTms;
});
```

---

## Step 3: Select the "Best Match" Entity

After sorting, the script selects the top result.

```bash
const bestMatch = entities[0];

return bestMatch.entityId;
```

---

## Outcome

This guarantees that Key Requests are always applied to the:

- Live service instance
- Most recently active entity
- Newest deployment version


---


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
