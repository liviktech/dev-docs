<!--
TEMPLATE for a developer implementation guide for a feature. Intended for developers and AI agents. Covers required fields, their originating database tables and UI modules, validation rules, and how to wire up or switch the implementation.

Copy this file into the feature folder and rename it to describe the feature (for example E_INVOICE_DEVELOPER_GUIDE.md or APPROVAL_WORKFLOW_DEVELOPER_GUIDE.md).

Fill in every <placeholder>. Keep sections even if the answer is short. If a section truly does not apply, write "None." instead of deleting it.

Delete all comment blocks before you finish. The final file should have no comments left.

Be specific. Name the actual file, database column, and module. Do not use vague placeholders like "service file" — write the real path.

Do not commit or push. This file is written to disk for documentation only.
-->

# <Feature Name> Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.
> **Purpose:** A concise reference detailing required fields, their originating modules and database columns, validation rules, and the exact steps to implement or modify <feature name>.

---

## 1. Required Fields & Originating Modules

<!--
Describe where the feature pulls its data from. Group by logical data source (for example: Company/Seller, Customer/Buyer, Line Items, Header). For each group, name the UI module where a user fills in the data and the database table/column where it is stored.
-->

To trigger <feature name>, the system aggregates data across <N> modules: **<Module 1>**, **<Module 2>**, and **<Module 3>**.

### 1.1 <Data Source Name> (for example: Company / Seller Details)
*Captured from <UI location or module> and stored on the <table> record.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **<Field Name>** | Yes | <UI location, for example: Settings / Profile Modal> | `<table>.<column>` | <Any constraint or format note> |
| **<Field Name>** | Yes | <UI location> | `<table>.<column>` | <Note> |
| **<Field Name>** | Optional | <UI location> | `<table>.<column>` | <Default value if not set> |

---

### 1.2 <Data Source Name>
*Captured from <UI location or module>.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **<Field Name>** | Yes | <UI location> | `<table>.<column>` | <Note> |
| **<Field Name>** | Yes | <UI location> | `<table>.<column>` | <Note> |
| **<Field Name>** | Yes | Auto-calculated | `<table>.<column>` | <Formula or derivation rule> |

---

### 1.3 <Data Source Name>
*Captured from <UI location or module>.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **<Field Name>** | Yes | <UI location> | `<table>.<column>` | <Note> |
| **<Field Name>** | Yes | Auto-derived | Derived | <How it is derived> |

<!-- Add or remove sections as needed. -->

---

## 2. Eligibility & Validation Rules Checklist

<!--
List every pre-condition the system checks before executing the feature. Number them. Be specific — name the exact field, value, or condition that must pass.
-->

Before running <feature name>, the system verifies:
1. **<Rule name>:** <Exact condition that must be true.>
2. **<Rule name>:** <Exact condition that must be true.>
3. **<Rule name>:** <Exact condition, including edge cases.>
4. **<Rule name>:** <Exact condition.>
5. **<Rule name>:** <Exact condition.>

---

## 3. Implementation Guide (optional — only include if this feature has a future implementation step)

<!--
Only keep this section if the feature has a known pending implementation step, for example: switching from a mock provider to a live API, wiring up a webhook, or enabling a flag. If the feature is fully implemented with no pending steps, delete this entire section (Section 3 and all its sub-steps).

Describe the architecture pattern used. Show a diagram of what is already done vs what needs to be changed. The goal is for a developer to know exactly which files to touch and which to leave alone.
-->

The implementation uses <describe the pattern, for example: the Provider Pattern, a service layer, a webhook handler>. <Describe what is already built vs what still needs work.>

To <implement / switch to live / configure>, only <N> areas need modification:

```text
┌─────────────────────────────────────────────────────┐
│               <UI or Frontend Layer>                │
│          (NO CHANGES NEEDED - COMPLETE)             │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│           <Service / Backend Layer>                 │
│          (NO CHANGES NEEDED - COMPLETE)             │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│     1. <File to edit> (EDIT: <what to change>)      │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│     2. <File to create or edit> (EDIT: <what>)      │
└──────────────┬──────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────┐
│     3. <Config or env file> (EDIT: <what>)          │
└─────────────────────────────────────────────────────┘
```

---

### Step 1: <First implementation step, for example: Configure Environment Variables>
Edit [`<path/to/file>`](<file:///absolute/path/to/file>):

```<language>
# <Comment describing what to set>
<CONFIG_KEY>=<value>
<CONFIG_KEY>=<value>
```

---

### Step 2: <Second step, for example: Implement the Provider or Service>
Create or edit [`<path/to/file>`](<file:///absolute/path/to/file>).

Implement the `<InterfaceName>` interface:

```typescript
// <Brief comment on what this class does>
export class <ClassName> implements <InterfaceName> {
  readonly name = '<ClassName>';

  async <methodName>(input: <InputType>): Promise<<ReturnType>> {
    // 1. Build the payload
    const payload = {
      // <map input fields to the expected shape>
    };

    // 2. Call the external API or internal service
    const response = await <callOrFetch>(payload);

    // 3. Handle errors
    if (!response.ok) {
      throw new <ErrorClass>(response.message);
    }

    // 4. Return the normalised result
    return {
      // <map response fields to the internal result shape>
    };
  }
}
```

---

### Step 3: <Third step, for example: Wire the implementation in the factory or router>
Edit [`<path/to/factory-or-router-file>`](<file:///absolute/path/to/file>):

```typescript
// <Brief comment on what to change and why>
import { <ClassName> } from './<file>.js';

export function get<Feature>(): <InterfaceName> {
  switch (env.<CONFIG_KEY>) {
    case '<LIVE_VALUE>':
      return new <ClassName>();
    default:
      return new <MockOrDefaultClass>();
  }
}
```

---

## 4. What Already Works (Zero Changes Needed)

<!--
List every part of the system that works automatically once the steps above are done. This tells the developer what they do NOT need to build and gives them confidence about the scope.
-->

When you make the above changes, the following work automatically without modification:

1. **<Component 1>:** <What it does automatically.>
2. **<Component 2>:** <What it does automatically.>
3. **<Component 3>:** <What it does automatically.>
4. **<Component 4>:** <What it does automatically.>

<!-- Add or remove as needed. -->
