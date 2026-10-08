# META FETCH TEST — Public Research Notes

> Public-friendly, redacted testing notes.
> Specific target identifiers, callback IDs, personal infrastructure, and sensitive test destinations have been generalized.

---

## Target Pattern

```http
GET /api/meta?url=<USER_CONTROLLED_URL>
Host: <TARGET_HOST>
```

Observed behavior:

```text
user controls URL
        ↓
target performs server-side fetch
        ↓
response metadata is parsed
        ↓
selected fields are returned as JSON
```

---

# 1. External Callback — Server-Side Fetch Confirmed

## Test

```text
https://<CONTROLLED_CALLBACK>/<ID>
```

Requested through:

```text
/api/meta?url=https://<CONTROLLED_CALLBACK>/<ID>
```

## Result

The controlled callback received an outbound `GET`.

Observed headers included Cloudflare Worker / Pages-related values.

The callback source did not match the researcher's normal client connection.

## Meaning

This confirmed that the URL was being fetched by server-side infrastructure rather than directly by the browser.

## Status

```text
CONFIRMED:
✓ server-side outbound request
✓ user-controlled external destination
✓ Cloudflare Worker / Pages fetch path
```

---

# 2. Plain HTTP and HTTPS

## Tests

```text
http://<CONTROLLED_CALLBACK>/<ID>
https://<CONTROLLED_CALLBACK>/<ID>
```

## Result

Both schemes triggered server-side requests.

## Status

```text
CONFIRMED:
✓ HTTP supported
✓ HTTPS supported
```

---

# 3. Redirect Following

## Setup

Controlled URL A returned:

```http
HTTP/1.1 302 Found
Location: https://<CONTROLLED_CALLBACK>/<URL-B>
```

The target was asked to fetch URL A.

## Result

Observed flow:

```text
GET URL A
↓
302 Location: URL B
↓
GET URL B
```

The second request was also made by the same server-side Worker path.

## Meaning

The fetcher automatically follows HTTP redirects.

## Status

```text
CONFIRMED:
✓ redirect following
✓ redirected destination fetched server-side
```

## Open Question

```text
Is destination validation repeated after redirects?
```

Not established.

---

# 4. Controlled Metadata Parsing

A controlled HTML page contained markers such as:

```html
<title>TITLE123</title>

<meta name="description" content="DESC456">
<meta property="og:title" content="OGTITLE111">
<meta property="og:description" content="OGDESC222">
<meta property="og:site_name" content="OGSITE333">
<meta property="og:image" content="https://<CONTROLLED_CALLBACK>/og-image.png">

<link rel="icon" href="https://<CONTROLLED_CALLBACK>/favicon-test.ico">
<link rel="apple-touch-icon" href="https://<CONTROLLED_CALLBACK>/apple-icon.png">
```

## Response

The endpoint returned selected values similar to:

```json
{
  "url": "https://<CONTROLLED_CALLBACK>/<PAGE>",
  "title": "TITLE123",
  "description": "DESC456",
  "image": "https://<CONTROLLED_CALLBACK>/og-image.png",
  "icon": "https://<CONTROLLED_CALLBACK>/apple-icon.png",
  "name": "OGSITE333"
}
```

## Parser Behavior Observed

```text
title
→ HTML <title>

description
→ meta name="description"

image
→ og:image

icon
→ apple-touch-icon

name
→ og:site_name
```

## Status

```text
CONFIRMED:
✓ fetched response body is parsed
✓ controlled metadata is returned in JSON
```

---

# 5. HTML-Looking Metadata

## Controlled Input

```html
<title>META-&lt;b&gt;TEST123&lt;/b&gt;</title>

<meta name="description"
      content="DESC-<b>TEST456</b>">
```

## Result

The API returned the markers as strings.

Observed difference:

```text
title
→ HTML entities remained encoded/preserved

description
→ HTML-looking text survived into JSON
```

## Frontend Check

The marker was not observed being rendered into a dangerous browser DOM sink during the tested wallet flow.

## Status

```text
CONFIRMED:
✓ HTML-looking metadata can survive parsing into JSON

NOT CONFIRMED:
✗ HTML injection
✗ XSS
✗ unsafe DOM rendering
```

## Decision

```text
No renderer found
→ no useful XSS chain demonstrated
→ park this branch unless another UI consumes the same fields
```

---

# 6. Upstream Error Response Parsing

## Test

A controlled endpoint returned an HTTP error response containing:

```html
<title>TEST404</title>
```

## Result

The metadata endpoint still returned a successful JSON response containing:

```json
{
  "title": "TEST404"
}
```

## Meaning

The metadata parser can inspect an upstream response body even when the upstream status is not `200 OK`.

## Status

```text
CONFIRMED:
✓ metadata parsed from upstream error response
```

This is behavior, not a vulnerability by itself.

---

# 7. Header Forwarding Check

The browser request to the target contained normal session/application context.

The outbound request received by the controlled callback did **not** include obvious target credentials such as:

```text
Cookie
Authorization
application-specific API key
session-specific authentication value
```

## Status

```text
CONFIRMED:
✓ no obvious sensitive credential forwarding observed
```

## Meaning

This reduced the likelihood of a simple credential-leak chain through arbitrary external destinations.

---

# 8. Fetching Another Public API

## Test

A publicly reachable API belonging to the same program was supplied as the metadata URL.

Example pattern:

```text
https://<PUBLIC_API>/<PUBLIC_RESOURCE>/<OWN_TEST_VALUE>
```

## Result

The fetch completed, but because the upstream resource returned JSON instead of HTML metadata, fields such as:

```text
title
description
image
name
```

were empty.

A favicon-style URL could still be derived from the supplied origin.

## Status

```text
CONFIRMED:
✓ another public program endpoint was reachable through the fetcher

NOT CONFIRMED:
✗ privileged access
✗ non-public data access
```

---

# 9. Non-Public Destination Checks

Some restricted/non-public destination forms were evaluated only to understand input handling.

The results did **not** establish successful access to internal services, cloud metadata, private network resources, or protected infrastructure.

Operational destination details are intentionally omitted from these public notes.

## Status

```text
NOT CONFIRMED:
✗ internal resource access
✗ private network access
✗ cloud metadata retrieval
✗ protected service access
```

## Meaning

Acceptance of a URL string is not the same as proving that the destination was actually reachable.

---

# 10. Relative Path / LFI-Style Input

## Test Pattern

```text
../../../../example
```

## Result

The endpoint rejected the value as an invalid URL.

## Meaning

The feature expects a syntactically valid absolute URL.

## Status

```text
NOT OBSERVED:
✗ LFI
✗ path traversal
```

---

# Confirmed Fetch Flow

```text
user controls URL
        ↓
<TARGET_HOST>/api/meta
        ↓
server-side Worker / function
        ↓
outbound HTTP(S) request
        ↓
optional redirect
        ↓
upstream response
        ↓
metadata parser
        ↓
selected fields returned as JSON
```

---

# Confirmed Capabilities

```text
✓ server-side external HTTP fetch
✓ server-side external HTTPS fetch
✓ redirect following
✓ controlled public destination/path
✓ upstream HTML parsing
✓ selected metadata extraction
✓ upstream error-body parsing
✓ public API destinations reachable
```

---

# Tested but NOT Demonstrated

```text
✗ internal/private resource access
✗ protected localhost/internal service access
✗ cloud metadata retrieval
✗ private network access
✗ credential or secret retrieval
✗ sensitive header forwarding
✗ LFI
✗ path traversal
✗ XSS
✗ HTML injection in the observed UI
```

---

# Current Classification

The strongest accurate statement from the testing is:

> The endpoint is a server-side URL-fetch and metadata-parsing feature. It accepts user-controlled HTTP(S) URLs, follows redirects, parses selected metadata from upstream responses, and returns those values as JSON.

This is **SSRF-shaped behavior**, but no meaningful security boundary was demonstrated during the tests.

---

# Why It Was Interesting

The feature immediately raises normal research questions:

```text
Does the server really fetch the URL?
Does it follow redirects?
Which schemes are supported?
What does it parse?
Are credentials forwarded?
Does it expose non-public resources?
Does returned metadata reach an unsafe frontend sink?
```

Those questions were worth testing.

The final result simply happened to be:

```text
interesting primitive
        ↓
characterized successfully
        ↓
no demonstrated impact
        ↓
document and pivot
```

---

# What Worked

```text
✓ proving the request was server-side
✓ proving redirect following
✓ proving HTTP + HTTPS behavior
✓ mapping metadata parser behavior
✓ proving upstream error bodies are still parsed
✓ checking credential forwarding
✓ testing whether controlled metadata reaches the UI
```

---

# What Did Not Produce Impact

```text
✗ metadata-to-XSS chain
✗ credential forwarding
✗ private/internal resource access
✗ arbitrary upstream body disclosure
✗ LFI/path traversal
```

---

# Research Decision

Do not keep throwing random payloads at a primitive once the important trust-boundary questions have been answered.

KISS decision tree:

```text
server-side fetch confirmed
        ↓
characterize behavior
        ↓
look for security boundary crossing
        ↓
no impact found
        ↓
save notes
        ↓
pivot to a higher-signal feature
```

---

# Remaining Architecture Questions

Potential future questions, if another feature provides a reason to revisit this primitive:

```text
Is destination validation applied before every redirect?

Are secondary metadata resources fetched server-side?

Does another application component consume these metadata fields?

Can the primitive be chained with another independently discovered weakness?
```

Until a meaningful boundary crossing is demonstrated, treat this as a documented server-side fetch primitive rather than a confirmed vulnerability.

---

# Final Note

A negative result is still useful research.

This test established:

```text
what the feature does
what it does not appear to do
which attack ideas were ruled out
where further testing stopped
why the researcher pivoted
```

That is the purpose of these notes.
