# API Document Template

Use this template when documenting a REST API endpoint.

## 1. API Specs

| Field | Value |
|-------|-------|
| Name | {endpoint name} |
| Description | {what this endpoint does} |
| URL | `https://{{host}}/{service}/v1/{resource}` |
| Method | {GET / POST / PUT / PATCH / DELETE} |
| Header | `Content-Type: application/json` |
| | `Idempotency-Key: {uuid}` (if applicable) |

## 2. Request Body

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| field_name | String | Yes | — | What this field represents. Constraints. Format. Example value. |

## 3. Response Body

**Success (200):**

```json
{
  "meta": {
    "code": "200000",
    "type": "SUCCESS",
    "message": "Success",
    "service_id": "{service-name}",
    "extra_meta": {}
  },
  "data": {
    "field_name": "value"
  }
}
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| field_name | String | Yes | — | What this field represents |

## 4. Errors

| Status Code | Code | Type | Description |
|-------------|------|------|-------------|
| 400 | 400001 | VALIDATION_ERROR | Request body failed validation |
| 401 | 401000 | UNAUTHORIZED | Missing or invalid authentication |
| 403 | 403000 | FORBIDDEN | Authenticated but not authorized |
| 404 | 404000 | NOT_FOUND | Resource does not exist |
| 409 | 409000 | CONFLICT | Resource state conflict |
| 500 | 500000 | INTERNAL_ERROR | Unexpected server error |

## 5. cURL Sample

```bash
curl --location --request POST 'https://example.com/{service}/v1/{resource}' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer {token}' \
  --header 'Idempotency-Key: {uuid}' \
  --data '{
    "field_name": "value"
  }'
```

## Notes

- Describe request and response body fields clearly with business logic, format constraints, and sample values.
- List all possible errors with their meanings.
- Always include a cURL sample for quick testing.
