# Injection and untrusted input

The fix for injection is always the same idea: keep data and code apart. Pass input as a parameter or argument to an API that treats it as data, instead of splicing it into text that something else parses.

## SQL

| Stack | Safe | Unsafe |
|---|---|---|
| Python DB-API (psycopg, mysqlclient) | `cur.execute("SELECT * FROM users WHERE email = %s", (email,))` | `cur.execute(f"... WHERE email = '{email}'")` |
| Python sqlite3 | `con.execute("SELECT * FROM users WHERE email = ?", (email,))` | `%` or f-string formatting into the SQL |
| SQLAlchemy | `select(User).where(User.email == email)`; `text("... WHERE email = :email")` with `{"email": email}` | `text(f"... {email}")` |
| Django | `User.objects.filter(email=email)`; `raw("... WHERE email = %s", [email])` | `raw(f"...")`, `extra(where=[f"..."])` |
| Node `pg` | `client.query('SELECT * FROM users WHERE email = $1', [email])` | `` client.query(`... '${email}'`) `` |
| Node `mysql2` | `conn.execute('SELECT * FROM users WHERE email = ?', [email])` | template strings |
| Prisma | ``prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}` `` (tagged template binds values) | `prisma.$queryRawUnsafe(...)` with input concatenated |
| Knex | `knex('users').where({ email })`; `knex.raw('... = ?', [email])` | `` knex.raw(`... ${email}`) `` |
| Go `database/sql` | `db.QueryContext(ctx, "SELECT ... WHERE email = $1", email)` (`?` for MySQL) | `fmt.Sprintf` into the query |
| Java JDBC | `PreparedStatement`, then `ps.setString(1, email)` | `"... WHERE email = '" + email + "'"` |
| Ruby ActiveRecord | `User.where(email: email)`; `User.where("email = ?", email)` | `User.where("email = '#{email}'")` |
| PHP PDO | `$st = $pdo->prepare('... WHERE email = ?'); $st->execute([$email]); $row = $st->fetch();` | string interpolation |
| C# / EF Core | `FromSql($"... WHERE email = {email}")` (interpolation becomes parameters); `SqlParameter` with ADO.NET | `FromSqlRaw("... '" + email + "'")` |

Identifiers (column names in `ORDER BY`, table names) can't be parameters. Map user choices to fixed identifiers:

```python
SORT_COLUMNS = {"name": "name", "created": "created_at"}
column = SORT_COLUMNS.get(requested_sort, "created_at")
```

In `LIKE` patterns, escape `%`, `_`, and the escape character in user input, then bind the pattern as a parameter.

## Shell and process execution

Pass a program and an argument list; never a command string through a shell.

| Language | Safe | Unsafe |
|---|---|---|
| Python | `subprocess.run(["git", "log", "--", path], check=True, capture_output=True, text=True)` | `os.system(f"git log {path}")`, any `shell=True` with input |
| Node | `execFile('git', ['log', '--', path])`, `spawn('git', ['log', '--', path])` | `` exec(`git log ${path}`) `` |
| Go | `exec.CommandContext(ctx, "git", "log", "--", path)` | `exec.Command("sh", "-c", "git log "+path)` |
| Ruby | `system("git", "log", "--", path)`, `Open3.capture2("git", "log", "--", path)` | backticks or one-string `system` with interpolation |
| Java | `new ProcessBuilder("git", "log", "--", path)` | `Runtime.exec("git log " + path)`, `sh -c` |
| PHP | `proc_open(['git', 'log', '--', $path], ...)` (PHP 7.4+) | `shell_exec("git log $path")`; use `escapeshellarg` only when no array form exists |

Argument injection still works without a shell: a value that starts with `-` becomes an option (`--upload-pack=...` for git, `-o` for many tools). Put `--` before user-supplied positional arguments, or reject values that start with `-`. Never let input choose the program itself.

## Path traversal

Resolve the final path and confirm it is inside the allowed root. Checking the input string for `..` misses encoded, absolute, and symlinked paths.

```python
from pathlib import Path

ROOT = Path("/srv/uploads").resolve()

def safe_path(name: str) -> Path:
    target = (ROOT / name).resolve()        # follows symlinks
    if not target.is_relative_to(ROOT):     # Python 3.9+
        raise PermissionError("path outside the upload root")
    return target
```

```js
const ROOT = path.resolve('/srv/uploads');

function safePath(name) {
  const target = path.resolve(ROOT, name);
  const rel = path.relative(ROOT, target);
  if (rel.split(path.sep)[0] === '..' || path.isAbsolute(rel)) {
    throw new Error('path outside the upload root');
  }
  return target; // path.resolve doesn't follow symlinks; use fs.realpath on existing files
}
```

```go
root, err := os.OpenRoot("/srv/uploads") // Go 1.24+: rejects paths that escape, including via symlinks
if err != nil {
	return err
}
defer root.Close()
f, err := root.Open(name)
```

```java
Path root = Paths.get("/srv/uploads").toRealPath();
Path target = root.resolve(name).normalize();
// normalize() doesn't follow symlinks: also check the real path of the file, or of its parent for a new file
Path existing = Files.exists(target) ? target : target.getParent();
if (!target.startsWith(root) || !existing.toRealPath().startsWith(root)) {
  throw new SecurityException("path outside the upload root");
}
```

Better still, don't use names from users for storage at all: generate an ID, store the original name as data, and look the file up by ID after checking ownership.

Archives: check every entry's path with the same function before extracting (zip slip), and cap the number of entries and the total uncompressed size (zip bombs). In Python, use `tarfile.extractall(filter="data")` (3.12+, also backported to 3.8.17, 3.9.17, 3.10.12, and 3.11.4).

## Server-side request forgery (SSRF)

When the server fetches a URL the user supplied (webhooks, link previews, image imports, PDF renderers):

1. Allow only `http` and `https`, and an allowlist of hosts when the use case permits.
2. Resolve the host name and reject the request if any resulting address is not public: loopback, private ranges, link-local (which includes `169.254.169.254`, the cloud metadata service), carrier-grade NAT (`100.64.0.0/10`), unspecified, multicast, and IPv4-mapped IPv6 versions of all of these.
3. Connect to the address you validated. Validating, then letting the HTTP client resolve the name again, leaves a DNS-rebinding gap.
4. Disable automatic redirects, or apply the same checks to every hop.
5. Set connect and read timeouts and a maximum response size. Don't return raw upstream errors or bodies to the caller.

In Go, a dialer `Control` function sees the actual address being dialed after resolution, which closes the rebinding gap:

```go
var cgnat = netip.MustParsePrefix("100.64.0.0/10")

func denyInternal(network, address string, _ syscall.RawConn) error {
	host, _, err := net.SplitHostPort(address)
	if err != nil {
		return err
	}
	ip, err := netip.ParseAddr(host)
	if err != nil {
		return err
	}
	ip = ip.Unmap()
	if !ip.IsGlobalUnicast() || ip.IsPrivate() || cgnat.Contains(ip) {
		return fmt.Errorf("blocked address %s", ip)
	}
	return nil
}

client := &http.Client{
	Timeout: 10 * time.Second,
	Transport: &http.Transport{
		Proxy:       nil, // a proxy would make the dialer see the proxy's address instead
		DialContext: (&net.Dialer{Control: denyInternal}).DialContext,
	},
	CheckRedirect: func(*http.Request, []*http.Request) error { return http.ErrUseLastResponse },
}
```

In Python, an address is acceptable when `ip.is_global and not ip.is_multicast` (`is_global` alone is true for multicast; unwrap `.ipv4_mapped` first for IPv6). Pinning the validated address with `requests` or `httpx` takes a custom transport, so for production use prefer an egress proxy that enforces the policy (Stripe's Smokescreen, for example) or a maintained SSRF-filtering library, and test it against `http://127.0.0.1`, `http://[::1]`, `http://169.254.169.254`, a host name that resolves to a private address, and a redirect to one.

## Cross-site scripting (XSS)

- Let the framework escape output. React, Vue, Angular, Svelte, Jinja2 (autoescape on), Django, ERB, Blade, Razor, and Go's `html/template` escape by default.
- Raw-HTML sinks must never receive user data unsanitized: `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `dangerouslySetInnerHTML`, `v-html`, Svelte `{@html}`, Handlebars `{{{ }}}`, Jinja `|safe` and `Markup()`, Django `mark_safe` and `|safe`, Rails `html_safe` and `raw`, Go `template.HTML`.
- When HTML from users is required (rich text), sanitize it with a maintained allowlist sanitizer: DOMPurify (browser), `sanitize-html` (Node), `nh3` (Python), `bluemonday` (Go), `rails-html-sanitizer` or `sanitize` (Ruby), OWASP Java HTML Sanitizer, HtmlSanitizer (.NET), HTML Purifier (PHP).
- URLs in `href` and `src`: parse them and allow only `http:`, `https:`, and `mailto:`; `javascript:` URLs run code even inside escaped attributes.
- Data embedded in a `<script>` block: use the framework's helper (Django `json_script`, `JSON.stringify` plus escaping `<` as `\u003c`), never string concatenation.
- Add a Content Security Policy as a second layer, not as the fix.

## Templates and code evaluation

Never treat user input as a template or as code: `render_template_string(user_input)`, `jinja2.Template(user_input)`, `Handlebars.compile(user_input)`, `eval`, `new Function`, `exec`, `setTimeout` with a string, and Node's `vm` module (not a security boundary). Pass user input as template variables instead.

## Deserialization

Only parse untrusted data with formats that can't construct arbitrary objects (JSON, then validate against a schema).

| Language | Dangerous with untrusted data | Use instead |
|---|---|---|
| Python | `pickle`, `marshal`, `shelve`, `jsonpickle`, `yaml.load` without a safe loader, `joblib.load`, `torch.load(weights_only=False)` | `json`; `yaml.safe_load`; `torch.load(..., weights_only=True)` (the default since PyTorch 2.6); safetensors for model weights |
| Java | `ObjectInputStream`, XStream without an allowlist, Jackson polymorphic typing (`activateDefaultTyping` with a permissive validator) | JSON into explicit DTOs |
| .NET | `BinaryFormatter` (removed in .NET 9), `NetDataContractSerializer`, Newtonsoft.Json with `TypeNameHandling` other than `None` | `System.Text.Json` into explicit types |
| PHP | `unserialize`, including with `allowed_classes => false` (PHP's manual warns against passing untrusted input to it at all) | `json_decode`; for data your own app stored, verify an HMAC before reading it |
| Ruby | `Marshal.load`, `YAML.unsafe_load` (and `YAML.load` before Psych 4) | `JSON.parse`; `YAML.safe_load` |
| Node | `node-serialize`, anything built on `eval` | `JSON.parse` plus schema validation (zod, ajv, valibot) |

## XML

Disable external entities and DTDs:

- Python: parse untrusted XML with `defusedxml`; with lxml, `etree.XMLParser(resolve_entities=False, no_network=True)`.
- Java: on `DocumentBuilderFactory`, `SAXParserFactory`, and `XMLReader`, `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`. On `XMLInputFactory` (StAX), `setProperty(XMLInputFactory.SUPPORT_DTD, false)` and `setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false)`. On `TransformerFactory`, `SchemaFactory`, and `Validator`, set `XMLConstants.ACCESS_EXTERNAL_DTD` and `ACCESS_EXTERNAL_STYLESHEET` (or `ACCESS_EXTERNAL_SCHEMA`) to `""`. The [OWASP XXE Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html) lists the settings for each parser.
- .NET: keep `DtdProcessing.Prohibit` (the default for `XmlReader`).
- PHP: libxml 2.9+ doesn't load external entities by default; never pass `LIBXML_NOENT`.

## Open redirects

Accept only local paths for `next`, `return_to`, and similar parameters:

```python
def safe_next(url: str) -> str:
    if (url.startswith("/") and not url.startswith(("//", "/\\"))
            and not any(c.isspace() or ord(c) < 32 for c in url)):
        return url
    return "/"
```

Browsers drop tabs and newlines from URLs, so `/\t/evil.example` becomes `//evil.example`; that's why the whitespace check matters. Prefer the framework's helper where one exists (Django's `url_has_allowed_host_and_scheme`).

## Mass assignment

Bind only the fields a caller may set: Rails `params.require(:user).permit(:name, :email)`; Django forms and DRF serializers with an explicit `fields` list (not `"__all__"` on models with sensitive fields); Pydantic input models with `model_config = ConfigDict(extra="forbid")`; zod schemas with `.strict()`; Laravel `$fillable`; separate request DTOs in Spring and ASP.NET. Fields such as `role`, `is_admin`, `owner_id`, `price`, `balance`, and `verified` should never come from the request body of an ordinary user.

## Prototype pollution (JavaScript)

Deep-merging untrusted JSON into objects can set `__proto__` or `constructor.prototype` and change every object in the process. Validate input with a schema that rejects unknown keys, use `Map` or `Object.create(null)` for dictionaries keyed by user input, keep merge utilities up to date, and consider `node --disable-proto=delete`.

## Regular expressions (ReDoS)

- Nested or overlapping quantifiers (`(a+)+`, `(a|aa)*`, `(.*,)*`) can take exponential time on crafted input. Cap the input length before matching.
- Engines with linear-time guarantees avoid the problem: Go's `regexp`, Rust's `regex` crate, RE2 bindings for other languages, and .NET's `RegexOptions.NonBacktracking` (.NET 7+). In .NET, also set a match timeout.
- Anchor validation patterns to the whole string. In Ruby, `^` and `$` match at line breaks; use `\A` and `\z`.

## Other injection points

- **CSV and spreadsheet formulas:** when exporting user data, prefix any cell that starts with `=`, `+`, `-`, `@`, tab, carriage return, line feed, or the full-width forms `＝ ＋ － ＠` with a single quote, so spreadsheet programs don't execute it. Then quote every cell and escape embedded quotes as the CSV format requires.
- **HTTP headers and email headers:** reject CR and LF in values placed in headers.
- **Logs:** use structured logging, or strip CR and LF from user-controlled values.
- **LDAP, XPath, NoSQL:** use the driver's parameter or filter-building APIs. For MongoDB, reject objects where a string is expected (`{"$ne": null}` in a login form is the classic attack) by validating types.
- **Numbers:** validate ranges and signs. Negative quantities, zero prices, and huge values break business logic; store money as integer minor units or decimals, never floats.
