# Input and injection

This document covers how to handle untrusted data. For language-specific dangerous functions, see `languages/<language>/security.md`. For file names and upload content, see [File handling](file-handling.md). For request body size and page size limits, see [Audit and abuse prevention](audit-and-abuse.md).

## Validation

### Validate all input crossing a trust boundary [Required]

This includes request parameters, headers, cookies, files, callbacks, and message queue content, as well as internal APIs, partner data, and records read back from storage. Validate on the server. Client-side validation is only for instant feedback.

**Why:** Client-side validation can be bypassed, and internal links do not guarantee data meets the receiver's rules either.

**Source:** OWASP Input Validation Cheat Sheet, OWASP ASVS

### Limit size first, then parse, then validate [Required]

**Applies to:** high risk

Limit request size before buffering or parsing, and configure a depth limit on the parser. Parse with a maintained parser, then validate the parsed result. Decode once per the protocol, validate the value actually used, and do not decode again downstream.

**Why:** Schema validation after parsing cannot protect the parser itself.

**Source:** OWASP Input Validation Cheat Sheet, OWASP ASVS

### Define allowlist rules per field [Required]

Define what is acceptable and reject everything else. Do not rely on a denylist to identify malicious strings.

| Input | Rule |
|---|---|
| Enum | Strictly within the allowed set |
| Numbers and dates | Type, format, minimum, maximum |
| Strings | Length limit, character or structure requirements |
| Objects | Required fields, allowed fields, handling of null |
| Arrays | Minimum and maximum element count, validate each item, including nested objects |

**Source:** OWASP Input Validation Cheat Sheet

### Explicitly reject unknown fields in schemas [Required]

Listing properties in JSON Schema does not require or restrict them. You must declare both `required` and `additionalProperties`. When a validation library ignores or strips unknown fields by default, enable strict mode or the equivalent configuration.

```json
{
  "type": "object",
  "required": ["sku", "quantity"],
  "additionalProperties": false,
  "properties": {
    "sku": { "type": "string", "maxLength": 32 },
    "quantity": { "type": "integer", "minimum": 1, "maximum": 99 }
  }
}
```

**Source:** OWASP Input Validation Cheat Sheet

### Validate business semantics [Required]

Beyond valid syntax, also validate semantics, for example that an end date is after the start date and that field combinations match. Run multi-step flows in the expected order, with no skipped steps.

**Source:** OWASP Input Validation Cheat Sheet, OWASP ASVS

### Reject on validation failure [Required]

Return an explicit error. Do not continue with partially validated data. Do not "clean" input and keep using it. Removing suspicious characters changes the meaning and does not do the job of parameterization and output encoding.

**Exceptions:** When page size exceeds the limit, handle it per [rest.md](../api/rest.md). Reject all other out-of-range numeric values.

**Source:** OWASP Input Validation Cheat Sheet

### Match the full value with regex and prevent ReDoS [Required]

**Applies to:** high risk

Require the whole value to match, limit length before matching, avoid patterns with excessive backtracking, and prefer non-backtracking engines or match timeouts.

**Source:** OWASP Input Validation Cheat Sheet, OWASP ASVS

### Compute key business values on the server [Required]

The server computes data such as prices, discounts, amounts, and stock from trusted data. Do not trust values sent by the client.

**Why:** Values sent by the client can be changed at will.

## General injection principles

### Separate data from instructions [Required]

Do not use string concatenation to put external data into any statement an interpreter can execute. Prefer that interpreter's parameterized interface. When there is none, use encoding specific to that context. Output encoding is the last step before handing off to the interpreter.

| Interpreter | Approach |
|---|---|
| SQL, HQL, NoSQL, graph queries | Parameterized queries |
| System commands | Library functions, or argument arrays |
| LDAP | Dedicated encoding functions |
| XPath | Parameterized or precompiled queries |
| Templates and expression languages | Fixed templates, data passed only as variables, not evaluated with `eval` |
| Email | Validate recipients and header fields to prevent injecting extra headers |
| HTML rich text | Use a maintained sanitization library |

**Source:** OWASP ASVS

## SQL injection

### Use parameterized queries [Required]

Always bind data from requests, other services, or storage as parameters. Raw SQL in ORMs and stored procedures must be parameterized too. Manual escaping depends on dialect and configuration and is not an alternative.

Avoid:

```text
"SELECT id FROM users WHERE email = '" + input + "'"
```

Prefer:

```sql
SELECT id FROM users WHERE email = ?;
```

**Tooling:** Semgrep

**Source:** OWASP SQL Injection Prevention Cheat Sheet, OWASP ASVS

### Use fixed mappings for dynamic structure [Required]

Placeholders can represent only values, not table names, column names, or `ASC`. For structure such as sort fields, map the external value to a fixed option in code, for example mapping `created` to `created_at`.

**Source:** OWASP SQL Injection Prevention Cheat Sheet

### Give database accounts least privilege [Recommended]

Application accounts do not use admin privileges and are granted only the tables and operations they need. This only limits the damage. It does not prevent injection.

**Source:** OWASP SQL Injection Prevention Cheat Sheet

## Command injection

### Prefer not to call system commands [Required]

When a library function can do the job, do not start an external process. When you must call one: take the command from a fixed allowlist, use an argument array instead of concatenating a shell string, validate each argument against an allowlist, and use `--` to end option parsing where supported.

**Example:** `["convert", "--", "/data/tmp/9f2c.png", "/data/tmp/out.png"]`

**Why:** Even without a shell, a value starting with `-` can still inject options.

**Source:** OWASP OS Command Injection Defense Cheat Sheet, OWASP ASVS

## Template injection

### Do not build templates from external input [Required]

Template content comes from code or controlled configuration, and external data is rendered only as variables. When users need custom templates, use a logic-limited template language or a strict sandbox.

**Example:** Do not write `render("Hello, " + input)`. Write `render("Hello, {{name}}", { name: input })`.

**Source:** OWASP ASVS

## Mass assignment

### Do not bind the request body directly to entities [Required]

Use a DTO or an explicit field list to restrict writable fields. Otherwise, users can send fields such as `role`, `isAdmin`, and `balance`.

**Source:** OWASP Mass Assignment Cheat Sheet, OWASP API Security Top 10

## Deserialization and XML

### Do not use native language serialization on untrusted data [Required]

Use a pure data format such as JSON. When you must use native serialization, restrict the allowed types, and verify a signature before deserializing. For specific functions, see the language layer.

**Why:** Native deserialization can lead to denial of service, privilege escalation, or remote code execution.

**Source:** OWASP Deserialization Cheat Sheet, OWASP ASVS

### Disable external entities in XML parsers [Required]

Disable DTDs where possible, and reject `DOCTYPE`. When DTDs are truly needed, disable external entities and external DTDs, and limit entity expansion. When a parser does not support a hardening setting, do not continue processing untrusted XML.

**Why:** XXE leads to local file reads, SSRF, and resource exhaustion.

**Source:** OWASP XXE Prevention Cheat Sheet, OWASP ASVS

## Exported content

### Prevent formula injection in CSV exports [Recommended]

When exporting to CSV or spreadsheets, escape per RFC 4180, and use a single quote to escape cells that start with `=`, `+`, `-`, `@`, a tab, or a null character.

**Why:** User data may be executed as formulas in spreadsheet software.

**Source:** OWASP ASVS
