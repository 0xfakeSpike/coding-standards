# File handling

This document covers uploads, paths, downloads, archives, and images. No single line of defense is enough, so combine these rules. For language-specific path APIs, see `languages/<language>/security.md`.

## Uploads

**Applies to:** high risk (all rules in this section)

### Define the boundaries of each upload feature first [Required]

For each upload entry point, state the allowed types, extensions, maximum size (including decompressed size), and how the file is handled on download. Complete authentication and authorization before the upload.

**Source:** OWASP ASVS, OWASP File Upload Cheat Sheet

### Validate type from content [Required]

After decoding the file name, validate the extension against an allowlist, then validate the type from the file content. Do not trust the extension or `Content-Type`. Allow only the types the business needs, and prefer low-risk types.

| Check | Stops | Limitation |
|---|---|---|
| Extension allowlist | `.php`, double extensions like `.jpg.php` | Does not mean the content is safe |
| `Content-Type` | Accidental uploads | Can be forged freely |
| File signature | Simple disguises | Easy to bypass, cannot be used alone |
| Parsing or rewriting with a dedicated library | Structurally invalid files | The parser itself also faces untrusted input |

**Why:** A denylist alone is easily bypassed with case changes, null bytes, or alternative extensions.

**Source:** OWASP File Upload Cheat Sheet, OWASP ASVS

### Regenerate file names [Required]

The server generates the storage name from a random value. Keep the original file name only as metadata, with limits on length and characters. When you must keep the user's file name, forbid leading dots and hyphens, consecutive dots, colons, and path separators.

**Why:** A file name can carry path traversal, overwrite an existing file, or reach scripts and HTML and cause injection.

**Source:** OWASP File Upload Cheat Sheet, OWASP ASVS

### Isolate storage and make it non-executable [Required]

Prefer a separate host or object storage for files, otherwise a location outside the web root. Upload directories are not executable, and file permissions are minimal. Prevent users from uploading directory configuration files such as `.htaccess` and `web.config`.

**Why:** Once the server can execute an uploaded script, that is remote code execution.

**Source:** OWASP File Upload Cheat Sheet, OWASP ASVS

### Limit size and total volume [Required]

Limit per-file size before reading into memory, and stream large files. Limit the total file count and total capacity per user to prevent storage exhaustion. Rate-limit both uploads and downloads.

**Source:** OWASP File Upload Cheat Sheet, OWASP ASVS

### Apply the same constraints to presigned uploads [Required]

When clients upload directly to object storage, the URL the server issues must be short-lived and restrict the object key, size, and type. Still validate the content after upload, and make the file visible externally only after it passes.

### Scan for malicious content in high-risk scenarios [Recommended]

Run files from untrusted sources through virus scanning before offering them for download. Provide a reporting channel for public content, and use manual review in a sandbox when necessary.

**Source:** OWASP ASVS, OWASP File Upload Cheat Sheet

## Path traversal

### Do not build paths from user input [Required]

Prefer mapping an internal ID to a server-generated path.

Avoid:

```http
GET /files?name=../../etc/passwd HTTP/1.1
```

Prefer:

```http
GET /files/7f3a9c HTTP/1.1
```

**Source:** OWASP ASVS

### Canonicalize and validate when you must use a user path [Required]

Decode first, resolve to a canonical absolute path, confirm it is still inside the allowed root, and only then access it. Compare by complete path segments, so `/data/uploads-evil` does not pass a prefix check for `/data/uploads`. Resolve symbolic links before deciding.

**Why:** Filtering only `../` is easily bypassed with encoded and nested forms.

**Source:** OWASP ASVS

## Downloads

### Check permissions on every download [Required]

File access goes through authorization checks. Do not rely on hard-to-guess paths. Signed download URLs are short-lived, and you authorize before generating them. Isolate multi-tenant paths by tenant. See [Authorization](authorization.md).

**Source:** OWASP Authorization Cheat Sheet, OWASP Multi-Tenant Security Cheat Sheet

### Make browsers treat files as attachments [Required]

**Applies to:** high risk

Serve user-uploaded content with `Content-Disposition: attachment`, specify a file name determined by the server, and add `X-Content-Type-Options: nosniff`. Encode the file name per RFC 6266 to prevent response header injection.

```http
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="report.pdf"; filename*=UTF-8''%E6%8A%A5%E5%91%8A.pdf
X-Content-Type-Options: nosniff
```

**Why:** When user-uploaded HTML or SVG renders on the business domain, its scripts can read site data.

**Source:** OWASP ASVS

### Serve user content from a separate domain [Recommended]

When online preview is needed, serve it from a domain different from the main site, and configure a restrictive CSP.

**Source:** OWASP ASVS

## Archives

### Prefer not to accept archives [Recommended]

Archives can contain files of any type, and there are many attack methods. When you must accept them, follow all rules in this section. Files such as `docx` and `odt` are also compressed formats.

**Source:** OWASP File Upload Cheat Sheet, OWASP ASVS

### Check totals before extracting [Required]

Before extracting, check the total decompressed size and file count. While extracting, count the bytes actually written and abort immediately when the limit is exceeded. Do not trust sizes declared in the archive header. Limit nesting depth.

**Why:** A compression bomb can exhaust disk and memory with a very small file.

**Source:** OWASP ASVS, OWASP File Upload Cheat Sheet

### Validate paths inside the archive when extracting [Required]

Treat paths inside the archive as untrusted. Canonicalize them and confirm they are inside the target directory, otherwise reject them (Zip Slip). Do not accept symbolic links. Extract to an isolated temporary directory, and validate the type of each file there before use.

**Source:** OWASP ASVS

## Image processing

**Applies to:** high risk (all rules in this section)

### Limit pixel dimensions before processing [Required]

Read width and height before decoding, and reject images over the limit outright. A very small file can decode into a huge bitmap.

**Source:** OWASP ASVS

### Re-encode and strip metadata [Recommended]

After decoding, re-encode to an allowed format and remove metadata such as EXIF, which may contain location information. Rewriting does not guarantee all malicious content is removed.

**Source:** OWASP File Upload Cheat Sheet, OWASP ASVS

### Isolate image processing [Recommended]

Image libraries parse untrusted input. Upgrade them promptly, run them in an isolated, resource-limited environment, and set timeouts and memory limits.

**Source:** OWASP File Upload Cheat Sheet

### Treat SVG as script content [Required]

SVG can contain scripts. Sanitize it to keep only safe tags and attributes, or convert it to a bitmap, or offer it only as an attachment download.

**Source:** OWASP ASVS
