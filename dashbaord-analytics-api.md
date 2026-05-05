# External custom fields API

API for letting customers define **custom fields** on top of unified models (e.g. `employee`, `candidate`) and map those fields to provider-specific paths in the upstream payload.

The unified model gives you a fixed schema (e.g. an `employee` always has `first_name`, `email`, etc.). Custom fields let a customer extend that schema with extra fields they care about — e.g. `guardian_mobile` — and tell us how to populate them by pointing a [JMESPath](https://jmespath.org/) expression at the raw upstream payload.

There are two scopes for a mapping:

- **Organization-scoped** (`integration_slug`) — applies to every connector of that integration in the org. Good for "every Workday connector should pull `guardian_mobile` from `data.employee.guardian_mobile`".
- **Connector-scoped** (`connector_token`) — applies to a single connector instance. Overrides the org-level mapping for that connector.

When both exist, **connector overrides organization**.

---

## Conventions

- **Base URL:** `/api/v1`
- **Auth:** `Authorization: Bearer <api_key>` on every request. All resources are scoped to the caller's organization.
- **Content type:** `application/json`
- **IDs:** UUIDs.
- **Errors:** standard `{"detail": "..."}` shape. `400` for validation, `401` unauth, `404` for missing resource.

Source files of interest:

- Routes: [app/router/common/v1/custom_field.py](../app/router/common/v1/custom_field.py)
- Lookups: [app/router/common/v1/lookup/](../app/router/common/v1/lookup/)
- Request schemas: [app/schemas/request/external/common.py](../app/schemas/request/external/common.py)
- Response schemas: [app/schemas/response/external/custom_field.py](../app/schemas/response/external/custom_field.py)

---

## Typical workflow

1. **Discover** — call the lookup endpoints to get valid `category`, `model`, and `integration_slug` values.
2. **Inspect upstream payload** — `GET /custom-fields/raw-data` to see the structure your JMESPath needs to navigate.
3. **(Optional) Validate JMESPath** — `POST /custom-fields/preview` to dry-run an expression before persisting.
4. **Create the field** — `POST /custom-fields`.
5. **Create a mapping** — `POST /custom-fields/mapping` (org- or connector-scoped).
6. **Verify per-connector view** — `GET /custom-fields/configuration` to see the effective mapping.

---

## Lookup endpoints

These power dropdowns / autocomplete in the UI when the user is creating a custom field or mapping. Returns only what the caller's org has access to.

### `GET /api/v1/lookup/models`

List unified models the org can target. Use `slug` as the `model` field elsewhere.

**Query params**
| name | type | required | notes |
| --- | --- | --- | --- |
| `category` | `Category` | no | e.g. `HRIS`. Filters to models in this category. |

**Response — `ModelListResponse`**

```json
{
  "models": [
    {
      "slug": "employee",
      "display_name": "Employee",
      "category": "HRIS"
    }
  ]
}
```

### `GET /api/v1/lookup/integrations`

List integrations enabled for the org. Use `slug` as `integration_slug` when creating org-scoped mappings.

**Query params**
| name | type | required | notes |
| --- | --- | --- | --- |
| `category` | `Category` | no | Filter to integrations enabled for this category. |

**Response — `IntegrationListResponse`**

```json
{
  "integrations": [
    {
      "slug": "workday",
      "display_name": "Workday",
      "categories": ["HRIS"]
    }
  ]
}
```

> Connector tokens (used for connector-scoped mappings) are **not** returned by a lookup — they are issued when a connector is created and surfaced through the connector flow.

---

## Custom field endpoints

### `POST /api/v1/custom-fields` — create a custom field

**Body — `CustomFieldCreateExternal`**
| field | type | required | notes |
| --- | --- | --- | --- |
| `name` | `string` | yes | snake*case, 2–128 chars. |
| `description` | `string` | no | |
| `category` | `Category` | yes | e.g. `HRIS`. |
| `model` | `string` | yes | Model slug from `/lookup/models`, max 64 chars, pattern `^[a-z]a-z0-9*-]\*$`. |

**`201` response**

```json
{
  "id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
  "status": "SUCCESS"
}
```

Fields created via this endpoint have `source = "API"`. Fields created from the dashboard have `source = "DASHBOARD"`.

---

### `GET /api/v1/custom-fields` — list custom fields

**Query params**
| name | type | notes |
| --- | --- | --- |
| `category` | `Category` | Optional filter. |
| `model` | `string` | Optional filter. |
| `source` | `"API" \| "DASHBOARD"` | Optional filter by origin. |
| `page_size` | `int` | Pagination. |
| `cursor` | `UUID` | Pagination. |

**`200` response — `PaginatedResponse[CustomFieldExternal]`**

```json
{
  "cursor": null,
  "page_size": 1,
  "items": [
    {
      "id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
      "name": "guardian_mobile",
      "description": "Employee's guardian mobile number",
      "category": "HRIS",
      "model": "employee",
      "source": "API",
      "created_at": "2026-05-01T10:00:00Z",
      "updated_at": null
    }
  ]
}
```

---

### `GET /api/v1/custom-fields/{custom_field_id}` — get a custom field with mappings

**`200` response — `CustomFieldDetailExternal`**

Same shape as `CustomFieldExternal` plus a `mappings` array (see mapping shape below).

```json
{
  "id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
  "name": "guardian_mobile",
  "description": "Employee's guardian mobile number",
  "category": "HRIS",
  "model": "employee",
  "source": "API",
  "created_at": "2026-05-01T10:00:00Z",
  "updated_at": null,
  "mappings": [
    {
      "id": "...",
      "custom_field_id": "...",
      "integration_slug": "workday",
      "connector_token": null,
      "json_path": "data.employee.guardian_mobile",
      "created_at": "...",
      "updated_at": null
    }
  ]
}
```

---

### `DELETE /api/v1/custom-fields/{custom_field_id}` — delete a custom field

Returns `204 No Content`. Cascades to its mappings.

---

## Mapping endpoints

A mapping pairs a custom field with a JMESPath expression, scoped to either an integration (org-level) or a single connector.

### `POST /api/v1/custom-fields/mapping` — create a mapping

**Body — `CustomFieldMappingCreateExternal`**
| field | type | required | notes |
| --- | --- | --- | --- |
| `custom_field_id` | `UUID` | yes | The field this mapping populates. |
| `integration_slug` | `string` | one-of | Set for org-scoped mapping. From `/lookup/integrations`. |
| `connector_token` | `string` | one-of | Set for connector-scoped mapping. |
| `json_path` | `string` | yes | JMESPath, max 1024 chars. |

> Provide **exactly one** of `integration_slug` or `connector_token`.

**`201` response**

```json
{
  "id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
  "status": "SUCCESS"
}
```

---

### `GET /api/v1/custom-fields/mapping` — list mappings

**Query params** — exactly one of:

- `integration_slug` — returns org-scoped mappings for that integration.
- `connector_token` — returns connector-scoped mappings for that connector.

**`200` response — `list[CustomFieldMappingExternal]`**

```json
[
  {
    "id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
    "custom_field_id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
    "integration_slug": "workday",
    "connector_token": null,
    "json_path": "data.employee.guardian_mobile",
    "created_at": "2026-05-01T10:00:00Z",
    "updated_at": null
  }
]
```

For a connector-scoped mapping `integration_slug` is `null` and `connector_token` is set; vice versa for org-scoped.

---

### `PATCH /api/v1/custom-fields/mapping/{custom_field_mapping_id}` — update mapping

Only `json_path` is mutable. Re-create the mapping if you need to change the scope.

**Body — `CustomFieldMappingUpdateExternal`**
| field | type | required |
| --- | --- | --- |
| `json_path` | `string` | yes |

**`200` response** — refreshed `CustomFieldMappingExternal` (same shape as list item).

---

### `DELETE /api/v1/custom-fields/mapping/{custom_field_mapping_id}` — delete mapping

Returns `204 No Content`.

---

## Discovery & validation endpoints

These help the user write a correct JMESPath without flying blind.

### `GET /api/v1/custom-fields/raw-data` — inspect raw upstream payload

Returns the raw JSON payload for a `(category, model)` so you can see what to JMESPath against.

**Query params** — `category` and `model` required, plus exactly one of:

- `connector_token` — returns the connector's latest synced row. Falls back to the integration's canned sample if it has never synced.
- `integration_slug` — returns the integration's canned sample (no connector required).

**`200` response** — raw JSON object (shape varies by integration). Example:

```json
{
  "data": {
    "employee": {
      "id": "emp_123",
      "first_name": "Ada",
      "guardian_mobile": "+1-555-123-4567"
    }
  }
}
```

---

### `POST /api/v1/custom-fields/preview` — dry-run a JMESPath

Evaluate a JMESPath against a connector's data **without** persisting it.

**Body — `CustomFieldPreviewRequest`**
| field | type | required | notes |
| --- | --- | --- | --- |
| `connector_token` | `string` | yes | |
| `category` | `Category` | yes | |
| `model` | `string` | yes | |
| `json_path` | `string` | yes | JMESPath to evaluate. |
| `raw_data` | `object` | no | Inline payload override. If omitted, the connector's latest synced row is used (falling back to integration sample). |

**`200` response — `CustomFieldPreviewResponse`**

```json
{
  "connector_token": "i5kqe8bSedEdPLX9pHhFojKdo7Uzvue0f7I4NVhaDfV0GTxF0uAgwa_COb3zZU3T",
  "json_path": "data.employee.guardian_mobile",
  "resolved_value": "+1-555-123-4567",
  "resolved_value_type": "string",
  "raw_data_source": "connector_sync"
}
```

`raw_data_source` will be one of:

- `"connector_sync"` — evaluated against the connector's latest synced row.
- `"integration_sample"` — connector hasn't synced yet, fell back to the canned sample.
- `"inline"` — `raw_data` was supplied in the request.

`resolved_value_type` is one of: `string`, `number`, `boolean`, `object`, `array`, `null`.

---

### `GET /api/v1/custom-fields/configuration` — effective view per connector

For a connector + `(category, model)`, return every custom field with the **effective** mapping in use (connector-scoped overrides org-scoped). Unmapped fields are included with `json_path: null` so callers can see what's still left to configure.

**Query params** (all required)
| name | type | notes |
| --- | --- | --- |
| `connector_token` | `string` | |
| `category` | `Category` | |
| `model` | `string` | |

**`200` response — `EffectiveCustomFieldResponse`**

```json
{
  "connector_token": "i5kqe8bSedEdPLX9pHhFojKdo7Uzvue0f7I4NVhaDfV0GTxF0uAgwa_COb3zZU3T",
  "integration_slug": "workday",
  "category": "HRIS",
  "model": "employee",
  "fields": [
    {
      "custom_field_id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734",
      "name": "guardian_mobile",
      "description": "Employee's guardian mobile number",
      "category": "HRIS",
      "model": "employee",
      "json_path": "data.employee.guardian_mobile",
      "source": "connector",
      "mapping_id": "018e586b-7d0b-7bc9-be65-a8fdbc82d734"
    },
    {
      "custom_field_id": "...",
      "name": "favorite_color",
      "description": null,
      "category": "HRIS",
      "model": "employee",
      "json_path": null,
      "source": null,
      "mapping_id": null
    }
  ]
}
```

Each entry's `source` indicates which mapping is winning:

- `"connector"` — a connector-scoped mapping is in effect.
- `"organization"` — no connector override; the org-scoped mapping is being inherited.
- `null` — no mapping is configured for this field on this connector.

---

## Endpoint summary

| Method   | Path                                  | Purpose                                              |
| -------- | ------------------------------------- | ---------------------------------------------------- |
| `GET`    | `/api/v1/lookup/models`               | List models for dropdowns.                           |
| `GET`    | `/api/v1/lookup/integrations`         | List integrations for dropdowns.                     |
| `POST`   | `/api/v1/custom-fields`               | Create custom field.                                 |
| `GET`    | `/api/v1/custom-fields`               | List custom fields.                                  |
| `GET`    | `/api/v1/custom-fields/{id}`          | Get custom field with mappings.                      |
| `DELETE` | `/api/v1/custom-fields/{id}`          | Delete custom field.                                 |
| `POST`   | `/api/v1/custom-fields/mapping`       | Create mapping (org- or connector-scoped).           |
| `GET`    | `/api/v1/custom-fields/mapping`       | List mappings by scope.                              |
| `PATCH`  | `/api/v1/custom-fields/mapping/{id}`  | Update mapping `json_path`.                          |
| `DELETE` | `/api/v1/custom-fields/mapping/{id}`  | Delete mapping.                                      |
| `GET`    | `/api/v1/custom-fields/raw-data`      | Inspect raw upstream payload for JMESPath authoring. |
| `POST`   | `/api/v1/custom-fields/preview`       | Dry-run a JMESPath.                                  |
| `GET`    | `/api/v1/custom-fields/configuration` | Effective per-connector view (overrides resolved).   |
