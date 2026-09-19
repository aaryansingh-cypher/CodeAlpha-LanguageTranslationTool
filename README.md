# Translate — Language Translation Tool

A single-page translation tool. Type text on the left, pick a source and target language, and read the translation on the right. Built as one self-contained HTML file: no build step, no dependencies, no framework.

---

## Features

| | |
|---|---|
| Language selection | 29 languages, plus automatic source-language detection |
| Live output | The translation streams in as it is produced, rather than appearing all at once |
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

## Connecting a translation API

The page ships with the translation call pointed at the Claude model made available to it by the host page, so it works out of the box when opened inside Claude.

To use **Google Cloud Translation** instead, replace the `translate()` call site in `translate.html`. The important rule: **the API key never goes in the browser.** Anyone can read client-side JavaScript, and a leaked key is billable to you. Put the key on a server you control and let the page call that server.

### 1. Client side

```js
async function translate(text, from, to) {
  const r = await fetch("/api/translate", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ q: text, source: from, target: to })
  });
  if (!r.ok) throw new Error("Translation failed");
  const data = await r.json();
  return data.translatedText;
}
```

Pass `source: ""` when the selector is set to **Detect language** — Google infers it and returns `detectedSourceLanguage`.

### 2. Server side (Node + Express)

```js
import express from "express";

const app = express();
app.use(express.json());

app.post("/api/translate", async (req, res) => {
  const { q, source, target } = req.body;

  const url = new URL("https://translation.googleapis.com/language/translate/v2");
  url.searchParams.set("key", process.env.GOOGLE_TRANSLATE_KEY);

  const upstream = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      q,
      target,
      ...(source ? { source } : {}),
      format: "text"
    })
  });

  const data = await upstream.json();
  if (!upstream.ok) return res.status(upstream.status).json({ error: data.error });

  const result = data.data.translations[0];
  res.json({
    translatedText: result.translatedText,
    detectedSourceLanguage: result.detectedSourceLanguage ?? source
  });
});

app.listen(3000);
```

Run it with the key in the environment, not in the source:

```bash
export GOOGLE_TRANSLATE_KEY="your-key-here"
node server.js
```

### Microsoft Translator

The same pattern applies. The endpoint is `https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=<code>`, the key goes in an `Ocp-Apim-Subscription-Key` header alongside `Ocp-Apim-Subscription-Region`, the body is `[{ "Text": "..." }]`, and the translation comes back at `data[0].translations[0].text`.

### Streaming

The current code renders partial output as it arrives. Google and Microsoft both return the finished translation in a single response, so with those backends the text simply appears at once — the `onText` handler can be dropped.

---

## How it works

**Layout.** A CSS grid holds two panels either side of a one-pixel spine. Below 720px the grid collapses to a single column, the spine is hidden, and `order` moves the swap control between the two stacked panels.

**Language data.** One `LANGS` array drives both dropdowns. Each entry carries a BCP-47 code, a display name, and an optional `rtl` flag. The code is reused for `SpeechSynthesisUtterance.lang`, so the browser picks a matching voice.

**Detection.** When the source is set to *Detect language*, a separate short request identifies the language from the first 400 characters and reports it under the output. It runs alongside the main translation, and stays silent if it fails — detection is a convenience, not a blocker.

**Cancellation.** Each request gets a fresh `AbortController`. Pressing **Stop** aborts it; any text already received stays on screen.

**Errors.** Failures are matched on a stable error code, not on message text. Recoverable problems (rate limits, expired sessions, oversized input) show a message and leave the button live; unrecoverable ones disable the button and say so once, instead of letting the user retry into the same wall.

**Speech.** `speechSynthesis` is feature-detected. The Listen button toggles between speaking and stopping, and cancels any in-progress utterance when the page unloads.

---

## Known limitations

- Voice quality and language coverage for **Listen** depend entirely on the voices installed on the user's device. Some languages will have no voice at all, and the button will do nothing for those.
- The 3,000-character cap is a client-side guard. Enforce a limit on the server too.
- There is no request throttling in the page. If you connect a paid API, add rate limiting server-side.
- Translation history is not saved; reloading clears the page.

---

## Possible extensions

- Persist recent translations in `localStorage` and show a history panel
- Detect the input language as the user types, with debouncing
- Add a "pronounce the source text" button alongside the existing one
- Show alternative translations or a word-by-word gloss for short phrases
- Support file upload (`.txt`) and download of the result
