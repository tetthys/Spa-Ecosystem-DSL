# ECOSYSTEM_DSL

This document defines the unified **SPA DSL (Domain-Specific Language)** used within the Tetthys SPA ecosystem.  
It provides a consistent, extensible, and reusable structure for exchanging data between server and client.  
All client-side logic that consumes SPA-related data should interpret the DSL according to this specification.

---

## 1. Design Goals

The DSL is designed with the following principles:

1. **Extensibility**  
   New domains—UI state, pagination, forms, resources—can be added without breaking the structure.

2. **Generality**  
   The DSL must describe navigation, view selection, form state, UI state, flash messages, and other backend-driven UI data in one unified container.

3. **Reusability**  
   Different parts of the application can interpret relevant DSL sections while sharing a single standard format.

4. **Stable interpretation**  
   Rules such as initial value priority (`old > request > injected > initial`) are clearly defined so that only adapters need modification if server-side formats change.

---

## 2. Top-Level DSL Structure

The complete DSL object has the following shape:

```ts
type SpaDsl = {
  navigation?: {
    action: "redirect" | "back";
    target?: string;
  } | undefined;

  view?: {
    key: string;
    props: Record<string, any>;
  } | undefined;

  payload: Record<string, any>;

  flash: Record<string, any>;

  meta: Record<string, any>;
};
````

Default empty DSL:

```json
{
  "navigation": undefined,
  "view": undefined,
  "payload": {},
  "flash": {},
  "meta": {}
}
```

---

## 3. Meaning of Top-Level Sections

### 3.1 `navigation`

Represents navigation intent.

```ts
type Navigation =
  | { action: "redirect"; target: string }
  | { action: "back" };
```

* `redirect` → Client resolves `target` using its own route map.
* `back` → Client performs history-style backward navigation.

### 3.2 `view`

A hint telling the client which view to render.

```ts
type View = {
  key: string;               // client resolves this using its view map
  props: Record<string, any>; // injected props for that view
};
```

Example:

```json
{
  "view": {
    "key": "UserIndex",
    "props": { "tab": "active" }
  }
}
```

### 3.3 `payload`

The main container for domain data:

* Form state
* UI state
* Pagination information
* Resource lists
* Any additional server-provided data

It is intentionally generic so that new namespaces can be added as needed.

### 3.4 `flash`

Short-lived, one-time messages.

```json
{
  "flash": {
    "message": "Saved successfully",
    "level": "success"
  }
}
```

Or multi-message:

```json
{
  "flash": {
    "messages": [
      { "type": "success", "text": "Saved" },
      { "type": "info", "text": "Profile updated" }
    ]
  }
}
```

### 3.5 `meta`

Non-UI metadata for tracing, logging, and debugging.

```json
{
  "meta": {
    "requestId": "req-123",
    "timestamp": "2025-11-27T10:00:00Z"
  }
}
```

---

## 4. Standard Payload Namespaces

Within `payload`, the following namespaces are considered standard:

* `payload.forms`
  Form states: values, validation, old/request/injected inputs.

* `payload.ui`
  UI state such as modal visibility, popover state, layout preferences.

* `payload.pagination`
  Pagination metadata for resources.

* `payload.resources`
  Server-provided lists: users, posts, products, etc.

Additional namespaces may be added as needed.

---

## 5. Form DSL: `payload.forms`

Each form is identified by a form key:

```ts
type SpaFormPayload = {
  old?: Record<string, any>;
  request?: Record<string, any>;
  injected?: Record<string, any>;
  validation?: Record<
    string,
    {
      value: any;
      is_error: boolean;
      messages: string[];
      origin?: string;
    }
  >;
};
```

Example:

```json
{
  "payload": {
    "forms": {
      "user.register": {
        "old": { "email": "old@example.com" },
        "request": { "email": "request@example.com" },
        "injected": { "email": "injected@example.com" },
        "validation": {
          "email": {
            "value": "invalid@example.com",
            "is_error": true,
            "messages": ["Invalid email."],
            "origin": "server"
          }
        }
      }
    }
  }
}
```

### 5.1 Initial Value Priority

For each field:

> **old > request > injected > initial (component-level)**

This rule is implemented in a central helper (e.g., `resolveInitialValues`) so that UI components remain clean and unaffected by backend schema changes.

---

## 6. UI State DSL: `payload.ui`

UI-related state is placed in `payload.ui`.

Example:

```json
{
  "payload": {
    "ui": {
      "isMainPopoverOpen": true,
      "activeModal": "userDetail"
    }
  }
}
```

UI states can grow freely inside this namespace.

Builder sugar is recommended:

```js
withUi({ isMainPopoverOpen: true });
```

Maps to:

```js
with({ ui: { isMainPopoverOpen: true } });
```

---

## 7. Pagination DSL: `payload.pagination`

Pagination data is placed inside `payload.pagination`.

Example:

```json
{
  "payload": {
    "pagination": {
      "users": { "page": 2, "perPage": 20, "total": 120 },
      "orders": { "page": 1, "perPage": 10, "total": 42 }
    }
  }
}
```

Builder sugar:

```js
withPagination({ users: { page: 2, perPage: 20, total: 120 } });
```

---

## 8. DSL Builder and Its Mapping

A builder constructs the DSL step-by-step:

```js
router
  .to("users.index")
  .view("UserIndex")
  .withViewProps({ tab: "active" })
  .with({ ui: { isMainPopoverOpen: true } })
  .withFlash({ message: "Welcome!" })
  .withMeta({ requestId: "req-123" })
  .send();
```

Mapping summary:

| Builder Method    | DSL Effect                       |
| ----------------- | -------------------------------- |
| `to(key)`         | `navigation.action = "redirect"` |
| `back()`          | `navigation.action = "back"`     |
| `view(key)`       | `view.key = key`                 |
| `withViewProps()` | merge into `view.props`          |
| `with()`          | merge into `payload`             |
| `withFlash()`     | merge into `flash`               |
| `withMeta()`      | merge into `meta`                |
| `build()`         | returns DSL without sending      |
| `send()`          | sends DSL to the transport layer |

Additional optional sugar:

| Sugar Method       | DSL Mapping                 |
| ------------------ | --------------------------- |
| `withUi(obj)`      | `with({ ui: obj })`         |
| `withPagination()` | `with({ pagination: obj })` |

---

## 9. Interpretation Rules (Client-Side)

Different parts of the client interpret relevant DSL portions:

* Navigation logic reads `navigation` and `view`.
* Form logic reads `payload.forms`.
* UI state logic reads `payload.ui`.
* Pagination logic reads `payload.pagination`.
* Flash logic reads `flash`.
* Logging/debugging reads `meta`.

Each domain is isolated and interprets only the fields it needs.

---

## 10. Extension Rules

When extending DSL capabilities:

### 10.1 Avoid adding new top-level keys

Prefer adding namespaces under `payload`:

```json
{
  "payload": {
    "notifications": [...],
    "filters": {...}
  }
}
```

### 10.2 Use descriptive namespace names

Examples of good patterns:

* `forms`
* `ui`
* `pagination`
* `resources`
* `filters`
* `notifications`

### 10.3 Centralize interpretation logic

Helpers should encapsulate rules such as:

* initial value resolution
* validation conversion
* payload → UI props transformation

---

## 11. Versioning Guidance

If DSL evolves:

* Adding fields → non-breaking.
* Renaming/removing fields → breaking.

Optional DSL version:

```json
{
  "meta": {
    "requestId": "req-123",
    "dslVersion": "0.2.0"
  }
}
```

Clients may support multiple versions if necessary.

---

This document defines the unified DSL standard for the Tetthys SPA ecosystem.
Additional domain-specific documents (e.g., form DSL, router behaviors) may be created on top of this base specification.