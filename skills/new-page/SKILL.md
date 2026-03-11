---
name: new-page
description: Scaffold a new Next.js page with DataTable, form, API hooks, types, and i18n. Use when the user wants to add a new frontend page for an entity.
argument-hint: <entity-name> [route-path]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Scaffold a New Frontend Page

Create a complete Next.js page for **$ARGUMENTS** following the exact project conventions.

## Argument Parsing

Parse `$ARGUMENTS` as: `<entity-name> [route-path]`
- entity-name: the backend entity (e.g., `FuelLog`, `crop-revenue`)
- route-path: optional App Router path (e.g., `farming/fuel-logs`). If omitted, infer from entity name.

Before generating, read the backend DTO/entity files to understand the exact fields and types. If no backend exists yet, ask the user for the field list.

## Steps

### 1. Add TypeScript types

**File:** `frontend/src/lib/types.ts` (append to existing file)

```typescript
// Entity type (matches backend Response DTO)
export interface EntityName {
  id: number;
  field: string;
  // ... all response fields
  createdAt: string;
  updatedAt: string;
}

// Request type (for form submissions)
export interface EntityNameRequest {
  field: string;
  optionalField?: string;
  // ... matches backend Request DTO
}
```

Conventions:
- Entity interface: all fields from Response DTO, dates as `string`, nullable as `Type | null`
- Request interface: optional fields use `?`, no id/timestamps
- Enums: `export type EnumName = "VALUE1" | "VALUE2" | ...;`

### 2. Add constants

**File:** `frontend/src/lib/constants.ts` (append to existing file)

```typescript
export const ENUM_VALUES = ["VALUE1", "VALUE2", ...] as const;
```

Only add if the entity has enums. Use `as const` arrays.

### 3. Add API hook

**File:** `frontend/src/hooks/useApi.ts` (append to existing file)

```typescript
export function useEntityNames(params?: string) {
  return useApi<import("@/lib/types").EntityName[]>(
    `/kebab-case-plural${params ? `?${params}` : ""}`
  );
}
```

Conventions:
- Function name: `use` + PascalCase plural (e.g., `useFuelLogs`)
- URL: kebab-case plural matching backend controller path (without `/api/v1` prefix — the api client adds it)
- Import type inline

### 4. Add i18n translations

**Files:** `frontend/src/messages/en.json` and `frontend/src/messages/hi.json`

Add a new section for the entity under the appropriate domain key, or create a new top-level key if it's a new domain. Include:
- Entity name (singular + plural)
- Field labels
- Form labels (add*, save*, new*)
- Any new enum values under `enums` key

Follow existing patterns in the translation files.

### 5. Create the page

**File:** `frontend/src/app/{route-path}/page.tsx`

Follow this exact structure:

```typescript
"use client";

import { useState } from "react";
import { useTranslations } from "next-intl";
import Button from "@/components/ui/Button";
import Card from "@/components/ui/Card";
import DataTable, { Column } from "@/components/ui/DataTable";
import ErrorAlert from "@/components/ui/ErrorAlert";
import Input from "@/components/ui/Input";
import Select from "@/components/ui/Select";
import DatePicker from "@/components/ui/DatePicker";
import { useEntityNames } from "@/hooks/useApi";
import { useDeleteHandler } from "@/hooks/useDeleteHandler";
import { useEditHandler } from "@/hooks/useEditHandler";
import { api } from "@/lib/api";
import type { EntityName, EntityNameRequest } from "@/lib/types";

const INITIAL_FORM: EntityNameRequest = { /* defaults */ };

export default function EntityNamesPage() {
  const t = useTranslations("domain");
  const tc = useTranslations("common");
  const te = useTranslations("enums");

  const { data, loading, error, refetch } = useEntityNames();
  const handleDelete = useDeleteHandler(
    "kebab-case-plural",
    tc("confirmDelete", { entity: t("entityPlural") }),
    tc("failedToDelete", { entity: t("entityPlural") }),
    refetch,
  );
  const handleEdit = useEditHandler<EntityName>("kebab-case-plural", refetch);
  const [showForm, setShowForm] = useState(false);
  const [form, setForm] = useState<EntityNameRequest>(INITIAL_FORM);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await api.post<EntityName>("/kebab-case-plural", form);
      setShowForm(false);
      setForm(INITIAL_FORM);
      refetch();
    } catch (err) {
      alert(err instanceof Error ? err.message : tc("failedToSave", { entity: t("entityPlural") }));
    }
  };

  const columns: Column<EntityName>[] = [
    // Text: { key: "name", header: tc("name"), editable: { type: "text" } }
    // Number: { key: "amount", header: tc("amount"), editable: { type: "number", step: "0.01" } }
    // Date: { key: "date", header: t("date"), editable: { type: "date" } }
    // Select: { key: "status", header: tc("status"), editable: { type: "select", options: statusOptions } }
    // Nullable render: { key: "notes", header: tc("notes"), render: (row) => row.notes ?? "-" }
    // Actions (always last):
    {
      key: "actions",
      header: tc("actions"),
      render: (row) => (
        <Button variant="danger" size="sm" onClick={() => handleDelete(row.id)}>
          {tc("delete")}
        </Button>
      ),
    },
  ];

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold text-gray-700 dark:text-gray-200">
          {t("entityPlural")}
        </h1>
        <Button onClick={() => setShowForm(!showForm)}>
          {showForm ? tc("cancel") : t("addEntity")}
        </Button>
      </div>

      {showForm && (
        <Card title={t("newEntity")}>
          <form onSubmit={handleSubmit} className="grid grid-cols-1 md:grid-cols-2 gap-4">
            {/* Input fields using Input, Select, DatePicker components */}
            {/* Required fields: add `required` prop */}
            {/* Number fields: add type="number" step="0.01" min="0" */}
            <div className="md:col-span-2">
              <Button type="submit" variant="save">{t("saveEntity")}</Button>
            </div>
          </form>
        </Card>
      )}

      <Card>
        {loading ? (
          <p className="text-gray-500 dark:text-gray-400">{tc("loading")}</p>
        ) : error ? (
          <ErrorAlert message={error} onRetry={refetch} />
        ) : (
          <DataTable<EntityName> columns={columns} data={data ?? []} onSave={handleEdit} />
        )}
      </Card>
    </div>
  );
}
```

### 6. Add sidebar navigation

**File:** `frontend/src/components/layout/Sidebar.tsx`

Add a nav link for the new page in the appropriate section. Read the file first to understand the existing structure.

### 7. Only import components that are actually used

- `Input` — for text and number fields
- `Select` — only if there are enum/select fields
- `DatePicker` — only if there are date fields
- `enumOptions` from constants — only if there are enum selects

## After Creating Files

1. Run lint: `cd frontend && npx eslint src/app/{route-path}/page.tsx --fix`
2. Verify it compiles: `cd frontend && npx tsc --noEmit`
3. Summarize what was created/modified with file paths
