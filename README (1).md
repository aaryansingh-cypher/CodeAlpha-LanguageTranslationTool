# Translate — Language Translation Tool

A single-page translation tool. Type text on the left, pick a source and target language, and read the translation on the right. Built as one self-contained HTML file: no build step, no dependencies, no framework.

---

## Features

| | |
|---|---|
| Language selection | 29 languages, plus automatic source-language detection |
| Direct API call | Translates via the Gemini API using a key you provide, stored only in your browser |
| Copy | One-click copy of the translated text, with a fallback for older browsers |
| Text-to-speech | "Listen" reads the translation aloud using the browser's built-in voices |
| Swap | Reverses the two languages and moves the translation into the input box |
| Right-to-left support | Output direction flips automatically for Arabic, Hebrew, Urdu and Persian |
| Keyboard | `Ctrl` / `⌘` + `Enter` translates; `Stop` cancels a request mid-flight |
| Limits | 3,000-character cap with a live counter |
| Theming | Follows the system light/dark setting |
| Accessibility | Visible keyboard focus, live region on the output, reduced-motion respected |

---

## Files

```
translate.html    The entire application — markup, styles, and script
README.md         This file
```

---

## Running it

Open `translate.html` in any modern browser, or serve the folder:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000/translate.html
```

The interface, language pickers, swap, counter, copy and speech all work immediately. The **Translate** button needs a translation backend — see below.

---

## Translation backend

The page calls the **Gemini API** directly from the browser (`gemini-2.0-flash`, `generateContent`). Click **API key** in the toolbar, paste a key from [Google AI Studio](https://aistudio.google.com/apikey), and **Save** — the key is kept only in that browser's `localStorage` and sent only to `generativelanguage.googleapis.com`.

This is fine for personal or local use. It is **not** appropriate for anything you deploy publicly: a key embedded in client-side JavaScript is visible to anyone who opens dev tools, and Google bills whoever holds the key. For a public deployment, proxy the request through your own server instead, and remove the client-side key field entirely.

### Proxying instead of a client-side key

Replace `callGemini()` in `translate.html` with a call to your own endpoint:

```js
async function callGemini(prompt, signal) {
  const r = await fetch("/api/translate", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    signal,
    body: JSON.stringify({ prompt })
  });
  if (!r.ok) throw { code: "upstream_error", message: r.statusText };
  const data = await r.json();
  return { text: data.text, truncated: false };
}
```

```js
// server.js — Node + Express
import express from "express";
const app = express();
app.use(express.json());

app.post("/api/translate", async (req, res) => {
  const url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent"
    + "?key=" + process.env.GEMINI_API_KEY;

  const upstream = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ contents: [{ role: "user", parts: [{ text: req.body.prompt }] }] })
  });

  const data = await upstream.json();
  if (!upstream.ok) return res.status(upstream.status).json({ error: data.error });
  res.json({ text: data.candidates[0].content.parts.map(p => p.text).join("") });
});

app.listen(3000);
```

```bash
export GEMINI_API_KEY="your-key-here"
node server.js
```

### Using Google Cloud Translation or Microsoft Translator instead

Either can replace Gemini the same way — swap `callGemini()` for a call to your backend, and have that backend call the provider.

**Google Cloud Translation** — server side:

```js
app.post("/api/translate", async (req, res) => {
  const { q, source, target } = req.body;
  const url = new URL("https://translation.googleapis.com/language/translate/v2");
  url.searchParams.set("key", process.env.GOOGLE_TRANSLATE_KEY);

  const upstream = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ q, target, ...(source ? { source } : {}), format: "text" })
  });

  const data = await upstream.json();
  if (!upstream.ok) return res.status(upstream.status).json({ error: data.error });
  const result = data.data.translations[0];
  res.json({ translatedText: result.translatedText, detectedSourceLanguage: result.detectedSourceLanguage ?? source });
});
```

**Microsoft Translator** — the endpoint is `https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=<code>`, the key goes in an `Ocp-Apim-Subscription-Key` header alongside `Ocp-Apim-Subscription-Region`, the body is `[{ "Text": "..." }]`, and the translation comes back at `data[0].translations[0].text`.

---

## How it works

**Layout.** A CSS grid holds two panels either side of a one-pixel spine. Below 720px the grid collapses to a single column, the spine is hidden, and `order` moves the swap control between the two stacked panels.

**Language data.** One `LANGS` array drives both dropdowns. Each entry carries a BCP-47 code, a display name, and an optional `rtl` flag. The code is reused for `SpeechSynthesisUtterance.lang`, so the browser picks a matching voice.

**API key.** The key lives in `localStorage` under `gemini_api_key`, set from the **API key** panel. It is read once at load and again whenever it's saved; nothing else on the page ever reads or writes it.

**Detection.** When the source is set to *Detect language*, a separate short request identifies the language from the first 400 characters and reports it under the output. It runs alongside the main translation, and stays silent if it fails — detection is a convenience, not a blocker.

**Cancellation.** Each request gets a fresh `AbortController` passed to `fetch`. Pressing **Stop** aborts it.

**Errors.** `callGemini()` maps HTTP and API-level failures onto a small set of codes (`bad_key`, `rate_limited`, `refused`, `empty_completion`, `upstream_error`) rather than showing raw response text. A bad key reopens the API-key panel; other failures just show a message and leave the button live to retry.

**Speech.** `speechSynthesis` is feature-detected. The Listen button toggles between speaking and stopping, and cancels any in-progress utterance when the page unloads.

---

## Known limitations

- The Gemini API key is stored in `localStorage` and sent straight from the browser. That's fine on your own machine, but anyone with access to that browser (or its dev tools) can read the key. Don't use this setup on a shared or public deployment — proxy through a server instead (see above).
- Voice quality and language coverage for **Listen** depend entirely on the voices installed on the user's device. Some languages will have no voice at all, and the button will do nothing for those.
- The 3,000-character cap is a client-side guard only.
- There is no request throttling in the page — nothing stops rapid repeated calls from burning through a key's quota.
- Translation history is not saved; reloading clears the page.

---

## Possible extensions

- Persist recent translations in `localStorage` and show a history panel
- Detect the input language as the user types, with debouncing
- Add a "pronounce the source text" button alongside the existing one
- Show alternative translations or a word-by-word gloss for short phrases
- Support file upload (`.txt`) and download of the result
