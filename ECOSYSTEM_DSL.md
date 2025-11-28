# **Tetthys SPA DSL Specification (Redesigned)**

*(Based on maintainability, structure, and UI DSL principles from the design report)*

---

# **Table of Contents**

1. **Introduction**
2. **1. Definition**

   * 1.1 Navigation
   * 1.2 View
   * 1.3 UI State
   * 1.4 Forms
   * 1.5 Pagination
   * 1.6 Resources
   * 1.7 Flash
   * 1.8 Meta
3. **2. Explanation**

   * 2.1 Convention Over Configuration
   * 2.2 Separation of Concerns
   * 2.3 Component-Based UI Architecture
   * 2.4 Developer Experience
4. **3. Examples**

   * 3.1 Redirect with view props
   * 3.2 Form validation example
   * 3.3 UI state example
   * 3.4 Paginated list example
5. **Conclusion**

---

# **Introduction**

This document defines the **unified, maintainable, extensible DSL** used across the Tetthys SPA ecosystem.
It incorporates the structural recommendations made in the design report, including strong conventions, clear schema boundaries, component-friendly separation, and long-term maintainability.

The DSL is a single JSON document sent from the backend to the frontend, expressing UI/navigation intent and providing structured UI data.

---

# **1. Definition**

The DSL is a standardized object containing eight well-defined sections:

```ts
type SpaDsl = {
  navigation?: Navigation | undefined;
  view?: View | undefined;
  ui: UiState;
  forms: FormsState;
  pagination: PaginationState;
  resources: ResourceMap;
  flash: FlashState;
  meta: MetaState;
};
```

A minimal DSL is:

```json
{
  "navigation": undefined,
  "view": undefined,
  "ui": {},
  "forms": {},
  "pagination": {},
  "resources": {},
  "flash": {},
  "meta": {}
}
```

---

## **1.1 Navigation**

```ts
type Navigation =
  | { action: "redirect"; key: string }
  | { action: "back" };
```

Represents the server’s navigation intention.

---

## **1.2 View**

```ts
type View = {
  key: string;
  props: Record<string, any>;
};
```

Determines which UI view to render and what props to inject.

---

## **1.3 UI State**

```ts
type UiState = {
  [key: string]: any;
};
```

Represents UI-layer state (modals, popovers, layout flags, etc.).

---

## **1.4 Forms**

```ts
type FormState = {
  old?: Record<string, any>;
  request?: Record<string, any>;
  injected?: Record<string, any>;
  validation?: ValidationMap;
};

type FormsState = {
  [formKey: string]: FormState;
};
```

**Value resolution priority**:

> **old > request > injected > initial(component)**

---

## **1.5 Pagination**

```ts
type PaginationState = {
  [resource: string]: {
    page: number;
    perPage: number;
    total: number;
  };
};
```

Standardized server-driven pagination metadata.

---

## **1.6 Resources**

```ts
type ResourceMap = {
  [resourceName: string]: any;
};
```

Holds server-provided domain objects or lists.

---

## **1.7 Flash**

```ts
type FlashState = {
  messages: Array<{ type: string; text: string }>;
};
```

One-time notification messages.

---

## **1.8 Meta**

```ts
type MetaState = {
  requestId?: string;
  timestamp?: string;
  dslVersion?: string;
};
```

Tracing and diagnostic information.

---

# **2. Explanation**

---

## **2.1 Convention Over Configuration**

The design report emphasizes predictable structure and strong conventions.
Thus, all DSL documents use fixed top-level keys:

```
navigation, view, ui, forms, pagination, resources, flash, meta
```

This removes ambiguity and simplifies tooling.

---

## **2.2 Separation of Concerns**

Each section has one job:

* **navigation** controls how to move
* **view** describes what to show
* **ui** represents client UI state
* **forms** handles field states/validation
* **pagination** is resource metadata
* **resources** carry domain data
* **flash** contains ephemeral messages
* **meta** includes diagnostics

This makes each subsystem trivial to implement and extend.

---

## **2.3 Component-Based UI Architecture**

The design report stresses UI componentization.
The redesigned DSL supports this by:

* letting server provide only **data**, not layout
* keeping view logic under `view.key + view.props`
* segregating form logic, ui logic, and content logic

This model is ideal for React-based component trees.

---

## **2.4 Developer Experience**

* DSL schema is simple and stable
* Extensions happen inside clear namespaces (`ui`, `forms`, etc.)
* `dslVersion` enables backward compatibility
* Easy to validate and test

This structure lowers cognitive overhead and improves reliability.

---

# **3. Examples**

---

## **3.1 Redirect with view props**

```json
{
  "navigation": {
    "action": "redirect",
    "key": "users.index"
  },
  "view": {
    "key": "UsersPage",
    "props": { "tab": "active", "layout": "card" }
  },
  "ui": {},
  "forms": {},
  "pagination": {},
  "resources": {},
  "flash": {},
  "meta": {
    "requestId": "abc-123",
    "dslVersion": "1.0.0"
  }
}
```

---

## **3.2 Form validation**

```json
{
  "navigation": {
    "action": "redirect",
    "key": "user.register"
  },
  "view": {
    "key": "UserRegisterView",
    "props": {}
  },
  "forms": {
    "user.register": {
      "old": { "email": "wrong@example.com" },
      "injected": { "country": "KR" },
      "validation": {
        "email": {
          "value": "wrong@example.com",
          "isError": true,
          "messages": ["Invalid email format"],
          "origin": "server"
        }
      }
    }
  },
  "ui": {},
  "resources": {},
  "pagination": {},
  "flash": {},
  "meta": {
    "requestId": "req-742",
    "timestamp": "2025-11-27T10:30:00Z"
  }
}
```

---

## **3.3 UI state update**

```json
{
  "view": {
    "key": "UserDetailView",
    "props": {}
  },
  "ui": {
    "modal": "userDetail",
    "modalArgs": { "userId": 14 }
  },
  "resources": {
    "users": [
      { "id": 14, "name": "Alice" }
    ]
  },
  "forms": {},
  "pagination": {},
  "flash": {},
  "meta": {}
}
```

---

## **3.4 Paginated list**

```json
{
  "view": {
    "key": "UserList",
    "props": {}
  },
  "pagination": {
    "users": {
      "page": 2,
      "perPage": 20,
      "total": 442
    }
  },
  "resources": {
    "users": [
      { "id": 1, "name": "A" },
      { "id": 2, "name": "B" }
    ]
  },
  "ui": {},
  "forms": {},
  "flash": {},
  "meta": {}
}
```

---

# **Conclusion**

This redesigned DSL:

* follows the structural and maintainability guidelines suggested in the report
* offers clear separation of concerns
* supports a modern component-based UI architecture
* remains stable while being highly extensible
* provides a predictable developer experience
* is fully future-safe with versioning and standardized namespaces
