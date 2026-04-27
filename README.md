# HTML Breakage Fuzzer

Passive Burp Suite extension for finding **HTML-context and JavaScript-context breakage** caused by reflected input.

Instead of trying to pop `alert()` or run active exploit payloads, this extension focuses on a safer question:

> **Does reflected input break the page structure or execution context in a way that could become exploitable?**

It watches in-scope traffic, mutates URL parameters with safe probes, and classifies the result into meaningful verdicts such as tag injection, attribute breakout, event-handler creation, and JavaScript string breakout.

---

## What it does

- Watches Burp traffic passively
- Only processes requests that are already in **Burp Target scope**
- Only fuzzes requests that have **URL/query parameters**
- Sends **one-parameter-at-a-time** mutations
- Uses **safe detection probes** instead of exploit payloads
- Reuses the original response as the baseline
- Ignores extension-generated traffic to avoid self-recursion
- Hides `NO_REFLECTION` noise completely
- Shows findings in a sortable dashboard
- Appends vulnerable findings to a text file
- Supports a configurable **rate limit** with a default of **5 requests/second**

---

## Why this matters

Many reflected-input issues are not obvious if you only check whether input appears in the response.

A value can be reflected:

- harmlessly as escaped text
- unsafely as raw text
- or in a way that **breaks HTML or JavaScript context**

This extension tries to separate those cases automatically.

Examples of dangerous breakage include:

- a reflected value creating a **real HTML element**
- a quote escaping an attribute and creating a **new attribute**
- a quote escaping into a **new event handler**
- a value breaking out of a quoted **JavaScript string**

These are the kinds of conditions that often lead to XSS if an attacker later supplies an execution payload. This extension is built to detect those conditions without depending on exploit execution.

---

## How findings become exploitable

This section is intentionally written at a **high level** for documentation purposes.

### 1. Tag injection

If reflected input becomes a real DOM node, the application is treating user input as HTML instead of text.

Example pattern:

```html
<div>search term</div>
```

becoming something structurally like:

```html
<div>search term <faique></faique></div>
```

That means user-controlled markup is entering the document structure.

### 2. Attribute breakout

If a quote escapes an attribute value, the attacker may be able to:

- inject a new benign attribute
- inject a dangerous attribute
- change how the browser parses the element

Example pattern:

```html
<input value="USER_INPUT">
```

becoming something structurally like:

```html
<input value="test" data-fqprobe="1">
```

or otherwise breaking the attribute boundary.

### 3. Event-handler creation

If reflection creates a new `on...` attribute, the context is especially dangerous because event-handler attributes are JavaScript execution sinks.

Example pattern:

```html
<a href="/x" title="USER_INPUT">
```

becoming structurally like:

```html
<a href="/x" title="test" onmouseover="...">
```

### 4. JavaScript string breakout

If reflected input appears inside a quoted JavaScript string and breaks out of that string, the page may be vulnerable even if HTML tags are encoded.

Example pattern:

```html
<script>
  var search = 'USER_INPUT';
</script>
```

If the user value can terminate the string early, the page has crossed from reflection into script-context breakage.

---

## Default detection probes

The extension ships with safe probes such as:

- `"><faique>`
- `'><faique>`
- `""><faique>`
- `<faique>`
- `</faique>`
- `" data-fqprobe="1`
- `' data-fqprobe='1`
- `"-1-"`
- `'-1-'`
- `"`
- `'`

These are intended to detect breakage, not demonstrate execution.

---

## Verdicts

| Verdict | Meaning |
|---|---|
| `VULNERABLE` | Injected markup became a real DOM element. |
| `VULNERABLE_ATTR_INJECTION` | Reflection escaped its attribute and created a new benign attribute. |
| `VULNERABLE_EVENT_HANDLER` | Reflection escaped its attribute and created a new event-handler attribute. |
| `VULNERABLE_JS_STRING_BREAK` | Reflection broke out of a quoted JavaScript string. |
| `VULNERABLE_ATTR_BREAK` | Reflection escaped its attribute context, but not in a more specific classified way. |
| `REFLECTED_RAW` | Input is reflected unescaped, but no structural breakage was confirmed. |
| `REFLECTED_ESCAPED` | Input is reflected in an encoded/escaped form. |
| `REQUEST_ERROR` | The fuzzed request failed or returned no usable response. |

`NO_REFLECTION` is intentionally **not shown** in the dashboard or logs.

---

## How it works

For each qualifying request:

1. The extension confirms the request is in **Burp Target scope**
2. It extracts URL parameters
3. For each parameter, it appends a random marker around one payload
4. It sends the mutated request, respecting the configured rate limit
5. It compares the mutated response against the original response
6. It classifies the result into one of the verdicts above

Important behavior:

- only **one parameter** is mutated per request
- the original response is used as the baseline
- internal extension requests are tagged and ignored by the observer
- `NO_REFLECTION` results are dropped before display/logging

---

## Installation

### Requirements

- Java 17+
- Gradle 8+
- Burp Suite with Montoya API support

### Build

```bash
cd burp_ext
gradle jar
```

Output:

```bash
build/libs/html-break-fuzzer-1.0.0.jar
```

### Load into Burp

1. Open **Burp Suite**
2. Go to **Extensions** → **Installed** → **Add**
3. Choose **Extension type: Java**
4. Select:

```text
build/libs/html-break-fuzzer-1.0.0.jar
```

5. A top-level tab named **HTML Breakage Fuzzer** will appear

---

## How to use

### 1. Add targets to Burp scope

The extension only works on requests already in **Burp Target scope**.

If a domain is not in scope, it will be ignored.

### 2. Configure settings

Open the extension tab and set:

- **Requests / second** — default is `5`
- **Payload list** — edit if you want custom probes
- **Output file** — where vulnerable findings are appended

### 3. Browse normally

As you browse the target through Burp:

- matching requests are fuzzed automatically
- findings appear in the dashboard
- vulnerable results are appended to the configured text file

### 4. Sort and review findings

The dashboard includes:

- numbered rows
- `#` column sorting
- verdict-based row colors
- vulnerable-only filter

---

## Example workflow

1. Put `https://target.example` in Burp Target scope
2. Browse the target through Burp Proxy
3. Visit pages that reflect search terms, IDs, names, filters, redirects, or state parameters
4. Watch the dashboard for:
   - `VULNERABLE`
   - `VULNERABLE_ATTR_INJECTION`
   - `VULNERABLE_EVENT_HANDLER`
   - `VULNERABLE_JS_STRING_BREAK`
   - `VULNERABLE_ATTR_BREAK`
5. Manually validate the context in Burp Repeater before reporting

---

## Rate limiting

The extension has a global fuzzing throttle:

- default: **5 requests/second**
- configurable in the Settings tab
- minimum: **1 request/second**

This helps reduce:

- WAF noise
- rate-limit bans
- accidental traffic spikes
- noisy self-testing against fragile endpoints

---

## Project structure

```text
src/main/java/com/faique/htmlbreak/
├─ HtmlBreakExtension.java
├─ ExtensionConfig.java
├─ BreakageHttpHandler.java
├─ FuzzerEngine.java
├─ HtmlAnalyzer.java
├─ Finding.java
├─ ResultsStore.java
├─ FileLogger.java
├─ DashboardPanel.java
├─ SettingsPanel.java
└─ MainTab.java
```

### File overview

- `HtmlBreakExtension.java` — Burp entry point
- `BreakageHttpHandler.java` — passive observer and gating logic
- `FuzzerEngine.java` — payload injection, throttling, request dispatch
- `HtmlAnalyzer.java` — DOM/script-context classification
- `DashboardPanel.java` — findings UI
- `SettingsPanel.java` — user configuration
- `ExtensionConfig.java` — persisted settings
- `FileLogger.java` — file output

---

## Safety notes

- Use only on systems you are authorized to test
- The extension sends additional requests automatically
- Findings indicate **dangerous parsing behavior**, not guaranteed exploitability in every case
- Always validate context manually before filing a report

---

## Suggested GitHub description

> Passive Burp Suite extension that detects HTML and JavaScript context breakage with safe probes, rate limiting, and verdict-based classification.
