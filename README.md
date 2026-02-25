# Logic Extensions

> ⚠️ **Beta**: Logic Extensions is currently in beta. APIs and behavior may change.

Logic Extensions is a plugin system that lets you inject custom logic into Arcade's execution flow. It's designed for organizations that need to integrate Arcade with their existing security, compliance, and identity management systems.

## Why Use logic Extensions?

When your organization adopts Arcade, you likely have existing systems that manage user permissions, security policies, and compliance requirements. Logic Extensions bridges Arcade with these systems, giving you:

- **Access Control** — Decide which users can see and use which tools based on your existing identity provider (Sailpoint, Entra, Okta, or custom systems)
- **Request Validation** — Check tool inputs against your policies before execution (e.g., "users can only send emails within their organization")
- **Output Filtering** — Modify or filter tool results before they reach the user (e.g., redact PII, scan for sensitive content)
- **Audit & Compliance** — Track all tool interactions through your security infrastructure

---

## Core Concepts

### Plugins

A **plugin** is a connection to an external system that makes decisions about access and execution. Think of it as the bridge between Arcade and your security infrastructure.

The most flexible plugin type is the **webhook plugin**, which lets you implement any custom logic by hosting HTTP endpoints that Arcade calls. Multiple of the same types of plugins can be configured and used together.

A plugin is configured once with its connection details (URLs, credentials) and can then be used across multiple hook configurations.

### Hook Points

**Hook points** are specific moments in Arcade's execution flow where your plugin can step in. The logic Extensions system is designed to support hook points across different parts of Arcade.

### Hook Configurations

A **hook configuration** connects a plugin to a specific hook point and defines how it should behave:

- Which plugin to use
- Which hook point to configure
- Execution priority (when multiple hooks exist)
- What to do if the plugin fails (block the operation or allow it to proceed)
- Filtering rules for when the hook applies

---

## Contextual Access

**Contextual Access** is the first logic Extensions feature, focused on tool execution. It provides three hook points that let you control who can access which tools and what happens before and after tools run.

### Hook Points

| Hook Point | When It Runs | What It Can Do |
|------------|--------------|----------------|
| **Access** | When listing available tools and before execution | Control which tools a user can see and use |
| **Pre-Execution** | Before a tool runs | Validate inputs, modify parameters, or block execution |
| **Post-Execution** | After a tool completes | Filter outputs, redact sensitive data, or block results |

### How It Works

#### Tool Discovery

When a user requests a list of available tools, Arcade checks with your Access hook to determine which tools they're allowed to see. This happens automatically and the user only sees tools they have permission to use.

#### Tool Execution

When a user runs a tool, the execution flows through your configured hooks:

```
User runs tool
      ↓
┌─────────────────┐
│     Access      │  → Can block if user isn't allowed to use this tool
└────────┬────────┘
         ↓
┌─────────────────┐
│  Pre-Execution  │  → Can validate inputs, modify parameters, or block based on request content
└────────┬────────┘
         ↓
┌─────────────────┐
│   Tool Runs     │
└────────┬────────┘
         ↓
┌─────────────────┐
│ Post-Execution  │  → Can filter outputs, redact data, or block results
└────────┬────────┘
         ↓
   Result returned
```

If any hook denies the request, the pipeline stops immediately and the user receives an error with the reason provided by your hook.

---

## Organization Structure

Hook configurations can be set at two levels:

### Organization Level

Organization-level configurations apply across your entire organization. Use these for:

- Company-wide compliance requirements
- Organization security policies
- Global audit logging

### Project Level

Project-level configurations apply only to a specific project. Use these for:

- Team-specific validation rules
- Project-level access restrictions
- Specialized data handling requirements

**Both levels always run.** A project-level configuration cannot bypass organization-level policies. This ensures company-wide compliance requirements are always enforced.

---

## Execution Order

Organization hooks have the option to run before or after project hooks to be the first or last validation.

When you have multiple hooks configured, they run in a predictable order:

1. **Organization hooks (before phase)** — Company-wide policies run first
2. **Project hooks** — Project-specific checks run next
3. **Organization hooks (after phase)** — Company-wide filtering and audit runs last

Within each phase, hooks run by priority order (lower numbers run first).

This structure ensures that:
- Organization policies are checked before project logic
- Final filtering (like PII redaction) happens after all other processing
- Audit logging captures the complete picture

### Hook Chaining

When multiple hooks are configured for the same hook point, they execute as a chain. Each hook receives the output of the previous hook, allowing transformations to build on each other.

```
Original Request
      ↓
┌─────────────────┐
│ Hook A (pri: 1) │  → Receives original request, returns modified request A
└────────┬────────┘
         ↓
┌─────────────────┐
│ Hook B (pri: 2) │  → Receives modified request A, returns modified request B
└────────┬────────┘
         ↓
┌─────────────────┐
│ Hook C (pri: 3) │  → Receives modified request B, returns final request
└────────┬────────┘
         ↓
   Final Request
```

#### Data Flow Rules

- **Pre-Execution hooks**: Each hook receives the tool inputs as modified by the previous hook. If Hook A enriches the input with additional parameters, Hook B sees those parameters.

- **Post-Execution hooks**: Each hook receives the tool output as modified by the previous hook. If Hook A redacts a field, Hook B never sees the original value.

- **Access hooks**: Each hook receives the full list of tools and returns the subset the user can access. The next hook only sees tools that passed the previous check.

#### Breaking the Chain

The chain stops immediately when any hook returns a **deny** response. Subsequent hooks in the chain do not execute.

```
┌─────────────────┐
│ Hook A (pri: 1) │  → Returns: ALLOW (modified request)
└────────┬────────┘
         ↓
┌─────────────────┐
│ Hook B (pri: 2) │  → Returns: DENY (reason: "blocked by policy")
└────────┬────────┘
         ✕
┌─────────────────┐
│ Hook C (pri: 3) │  → Never executes
└─────────────────┘
         ↓
   Error returned to user: "blocked by policy"
```

#### Priority Assignment

Priority is a numeric value where **lower numbers execute first**.

#### Cross-Phase Behavior

The three phases (org-before, project, org-after) are independent chains. The output of the org-before phase becomes the input to the project phase, and the project phase output becomes the input to org-after.

```
┌──────────────────────────────────────────────────────┐
│                   ORG-BEFORE PHASE                   │
│  Hook 1 → Hook 2 → Hook 3                            │
└──────────────────────┬───────────────────────────────┘
                       ↓ (modified request)
┌──────────────────────────────────────────────────────┐
│                    PROJECT PHASE                     │
│  Hook A → Hook B                                     │
└──────────────────────┬───────────────────────────────┘
                       ↓ (further modified request)
┌──────────────────────────────────────────────────────┐
│                   ORG-AFTER PHASE                    │
│  Hook X → Hook Y                                     │
└──────────────────────────────────────────────────────┘
```

If any hook in any phase denies the request, the entire pipeline stops and subsequent phases do not execute.

---

## Common Use Cases

### Entitlement-Based Access Control

Connect Arcade to your identity provider to control tool access based on user roles and entitlements. Only users with the appropriate permissions see and use specific tools.

### Input Validation

Enforce business rules before tools execute:
- Ensure emails can only be sent to approved domains
- Validate that data queries stay within permitted boundaries
- Block operations on sensitive resources

### Output Filtering

Process tool results before they reach users:
- Redact personally identifiable information (PII)
- Scan for and block sensitive content
- Transform data to meet compliance requirements

### Audit Trail

Log every tool interaction for compliance:
- Track who used which tools and when
- Record what inputs were provided
- Capture tool outputs for review

### Input Enrichment

Modify tool inputs before execution:
- Inject user-specific API keys or credentials
- Add contextual information the user doesn't have access to
- Transform identifiers to internal formats

---

## Failure Handling

Each hook configuration specifies what happens if the external system is unavailable:

| Mode | Behavior | Best For |
|------|----------|----------|
| **Fail Closed** | Block the operation | Security-critical checks where you can't afford to skip validation |
| **Fail Open** | Allow the operation | Non-critical enhancements like metrics or optional logging |


