# Python security

For general backend security, see [backend security rules](../../backend/index.md). This document covers only Python-specific risks.

## Language-specific risks

### Do not use eval and exec on external input [Required]

`eval`, `exec`, and `compile` take no external data. Parse literals with `ast.literal_eval` and data with `json`.

**Why:** These functions execute arbitrary code with the privileges of the current process.

**Tooling:** `S307`, `S102`.

**Source:** Ruff Docs, Bandit Docs

### Do not concatenate external input into SQL or templates [Required]

Do not build SQL with f-strings, `%`, or `format`. Use the parameter binding of the driver or ORM. Likewise, do not turn off `autoescape` in Jinja2. For the related Django rules, see [django.md](django.md).

**Example:**

Avoid:

```python
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
```

Prefer:

```python
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

**Tooling:** `S608`, `S701`.

**Source:** Ruff Docs, Bandit Docs

### Handle files safely [Required]

Pass `filter="data"` explicitly to `tarfile`'s `extractall`. For temporary files, use `NamedTemporaryFile`, `mkstemp`, or `TemporaryDirectory`. Do not use `tempfile.mktemp` or write a fixed `/tmp` path. Handle user-provided paths according to [file-handling.md](../../backend/security/file-handling.md). `Path.resolve()` resolves symbolic links.

**Why:** A malicious archive can use path traversal to overwrite files outside the target directory. `mktemp` leaves a time window between generating the file name and creating the file.

**Tooling:** `S202`, `S306`, `S108`.

**Source:** Python Standard Library Docs

### Set timeouts and verify certificates on HTTP calls [Required]

Pass `timeout` explicitly to `requests` calls, which never time out by default. Do not write `verify=False`.

**Tooling:** `S113`, `S501`.

**Source:** Requests Docs, Ruff Docs

### Confirm the Expat version before parsing untrusted XML [Required]

Attacker-controlled XML can be used for denial of service. Use `pyexpat.EXPAT_VERSION` to confirm that the runtime's Expat has fixed known issues, or switch to JSON. `xmlrpc` carries a decompression bomb risk.

**Tooling:** `S314` and the rules in the same group.

**Source:** Python Standard Library Docs

## Random numbers

### Use secrets for security purposes [Required]

Use `secrets` for tokens, passwords, and verification codes, for example `secrets.token_urlsafe(32)`. Use `random` only for simulations and test data. Compare tokens and signatures with `secrets.compare_digest`, not `==`, whose timing may be exploited in a timing attack.

**Tooling:** `S311`.

**Source:** Python secrets Docs

### Use dedicated algorithms for passwords [Required]

For algorithm order and parameters, see [authentication.md](../../backend/security/authentication.md). Use a dedicated password library for Argon2id. Do not use MD5 or SHA-1 for security purposes. When using them only as checksums, pass `usedforsecurity=False`.

**Tooling:** `S324`.

**Source:** Python Standard Library Docs

## Serialization

### Do not deserialize untrusted data [Required]

Do not process untrusted data with `pickle`, `shelve`, or `marshal`. Use JSON across trust boundaries. When pickle is truly needed, sign it with `hmac` to prevent tampering. For YAML, use `yaml.safe_load`, not `yaml.load` with the default loader.

**Why:** Loading malicious pickle data executes arbitrary code, and unsafe YAML loaders can instantiate arbitrary objects.

**Tooling:** `S301`, `S403`, `S506`.

**Source:** Python pickle Docs, Ruff Docs

## Command execution

### Call subprocesses with argument lists [Required]

Pass `subprocess` an argument list. Do not enable `shell=True` or use `os.system`. Put `--` before external file names so they are not parsed as options. Write `check=True` and `timeout`. On timeout, the child process is killed and `TimeoutExpired` is raised.

**Why:** Without a shell, metacharacters pass to the child process as-is. With the shell enabled, escaping is entirely the application's responsibility.

**Example:**

Avoid:

```python
subprocess.run(f"convert {filename} out.png", shell=True)
```

Prefer:

```python
subprocess.run(["/usr/bin/convert", "--", filename, "out.png"], check=True, timeout=30)
```

**Tooling:** `S602`, `S604`, `S605`.

**Source:** Python subprocess Docs

### Use full paths for executables [Recommended]

Use absolute paths, or resolve the path first with `shutil.which`, because `PATH` lookup behavior varies by platform. On Windows, the system may hand `.bat` and `.cmd` files to the shell even without `shell=True`, and Python does not escape the arguments. Do not pass external arguments to them.

**Tooling:** `S607`.

**Source:** Python subprocess Docs

## Static scanning

### Run security scans in CI [Recommended]

**Applies to:** team project

Use Ruff's `S` rule group for routine checks, or `bandit -c pyproject.toml -r src` for a standalone scan. Their rules overlap, so pick one. When suppressing, state the rule number, for example `# nosec B602` or `# noqa: S603`. Otherwise, issues added to the same line later are hidden too.

**Tooling:** `PGH004`.

**Source:** Bandit Docs, Ruff Docs

### Scan dependencies for vulnerabilities with pip-audit [Recommended]

**Applies to:** team project

CI runs `pip-audit -r requirements.txt` on locked dependencies. It queries vulnerability data from PyPI or OSV. uv projects export first with `uv export`.

**Source:** pip-audit Docs
