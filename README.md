# Gemini AI Clone

A fully functional clone of Google's Gemini AI chat interface — built with React 19, Vite, and the official Google Generative AI SDK. The app connects to the real Gemini 2.0 Flash model via API, streams responses word by word with a typing animation, maintains a session-level prompt history in the sidebar, and replicates Gemini's exact visual layout — collapsible sidebar, gradient logo, animated loading skeleton, and a fixed bottom input bar with conditional send button logic.

---

## What This Project Does

A user types any prompt into the input bar and clicks the send icon (which only appears when the input field is non-empty). The prompt is sent to Gemini 2.0 Flash via the `@google/generative-ai` SDK. While the response is loading, an animated 4-bar shimmer skeleton replaces the result area. Once the response arrives, it is parsed to convert Gemini's `**bold**` markdown into `<b>` HTML tags and `*` bullet markers into `</br>` line breaks — then the cleaned text is rendered word by word with a 75ms delay between each word, producing a live typing effect. Every prompt sent during the session is stored in the sidebar's recent history and can be clicked to re-run that prompt instantly.

---

## Architecture — How the App Is Structured

The app uses React Context (`Context.jsx`) as the single source of truth for all state and logic. Both `Main.jsx` and `Sidebar.jsx` consume this context via `useContext(Context)`. No prop drilling anywhere.

**State managed in `Context.jsx`:**

| State variable | Type | Purpose |
|---|---|---|
| `input` | string | Current value of the text input field |
| `recentPrompt` | string | The prompt that was most recently sent — displayed above the AI response |
| `prePromt` | array | Growing list of all prompts sent this session — powers the sidebar history |
| `showResult` | boolean | `false` = show the greeting screen; `true` = show the result panel |
| `loading` | boolean | `true` while the API call is in-flight — triggers the shimmer loader |
| `resultData` | string | The formatted, word-by-word-built response string rendered as HTML |

---

## How the API Integration Works — `src/conflig/Gemini.js`

The SDK is initialized with the API key from the `.env` file:

```js
const apikey = import.meta.env.VITE_GEMINI_API_KEY;
const genAI = new GoogleGenerativeAI(apikey);
const model = genAI.getGenerativeModel({ model: 'gemini-2.0-flash' });
```

The model used is **`gemini-2.0-flash`** — Google's fast, efficient model. Generation is configured with:

```js
const generationConfig = {
  temperature: 1,
  top_p: 0.95,
  top_k: 64,
  maxOutputTokens: 8192,
  responseMimeType: "text/plain",
};
```

`temperature: 1` means full creative randomness. `maxOutputTokens: 8192` allows long responses. Each call starts a fresh chat session with `history: []` — there is no multi-turn conversation memory between prompts; every prompt is stateless.

The `run(prompt)` function starts a new `chatSession`, sends the prompt via `sendMessage()`, and returns the plain text response string.

---

## How `onSent()` Works — The Core Flow

`onSent(prompt)` in `Context.jsx` is the function called on every prompt submission:

**Step 1 — Reset state for the new response:**
```js
setResultData("");
setLoading(true);
setShowResult(true);
```

**Step 2 — Determine the prompt source:**
If `onSent` is called from the sidebar's recent history (`loadPrompt`), it receives an explicit `prompt` argument. If called from the input bar, `prompt` is `undefined` and the current `input` state is used instead. In the input-bar case, the prompt is also pushed into the `prePromt` history array:
```js
setPrePrompt((prev) => [...prev, input]);
```

**Step 3 — Call the API:**
```js
response = await run(prompt || input);
```

**Step 4 — Parse Gemini's markdown into HTML:**
Gemini wraps bold text in `**double asterisks**`. The response is split on `**`, and every odd-indexed segment (inside the asterisks) is wrapped in `<b>` tags:
```js
let responseArray = response.split("**");
for (let i = 0; i < responseArray.length; i++) {
  newResponse += (i % 2 !== 1) ? responseArray[i] : "<b>" + responseArray[i] + "</b>";
}
```
Then every remaining `*` (single asterisk, used for bullet points) is replaced with `</br>`:
```js
let newResponse2 = newResponse.split("*").join("</br>");
```

**Step 5 — Word-by-word typing animation:**
The cleaned response is split on spaces into a word array. Each word is scheduled via `setTimeout` with a delay of `75 * index` milliseconds:
```js
delaypara(i, nextWord + " ");
// Inside delaypara:
setTimeout(() => setResultData((prev) => prev + nextWord), 75 * index);
```
This builds the `resultData` string one word at a time — the component re-renders on every `setResultData` call, producing the typing effect. A 500-word response takes approximately 37.5 seconds to fully type out.

**Step 6 — Cleanup:**
```js
setLoading(false);
setInput("");
```

---

## Components

### `Main.jsx` — The Chat Interface

The main panel consumes `input`, `setInput`, `recentPrompt`, `showResult`, `loading`, `resultData`, and `onSent` from context.

**Greeting screen** (`showResult === false`):
Shows "Hello, Pawan" in a green-to-teal gradient and "How can I help you today?" in gray beneath it. This is the default state before any prompt is sent.

**Result panel** (`showResult === true`):
- Displays the `recentPrompt` (the sent question) next to a `<AccountCircleIcon>` user avatar
- If `loading === true`: renders 4 `.load` divs — a shimmer skeleton animated with a pink-blue-purple gradient sweeping left to right (`background-size: 800px 50px`, `animation: loader 3s linear infinite`)
- If `loading === false`: renders the `resultData` string as raw HTML via `dangerouslySetInnerHTML={{ __html: resultData }}` — necessary because the response contains `<b>` and `</br>` tags

**Input bar:**
A pill-shaped div (`.search-box`) containing a text input, `<ImageSearchIcon>`, `<MicIcon>`, and conditionally the `<SendIcon>`. The send icon only renders when `input` is non-empty:
```jsx
{input ? <SendIcon onClick={() => onSent()} /> : null}
```
The image and mic icons are present in the UI but are not wired to any functionality — they are visual placeholders matching Gemini's interface.

**Bottom disclaimer:**
"Gemini may display inaccurate info..." — the exact same disclaimer text shown in Google's real Gemini interface.

---

### `Sidebar.jsx` — Navigation and History

The sidebar has two sections — top and bottom — with a collapsible toggle.

**Collapse/expand toggle:**
Local `useState(false)` controls `extended`. Clicking the `<MenuIcon>` toggles `extended`. When `extended === false`, only icons are visible. When `extended === true`, text labels and the recent history list appear.

**New Chat button:**
Calls `newChat()` from context, which resets `loading` to `false` and `showResult` to `false` — returning the UI to the greeting screen without clearing the input or history.

**Recent history list:**
Renders the `prePromt` array. Each entry shows the first 18 characters of the prompt followed by "..." — matching Gemini's sidebar truncation behavior. Clicking any entry calls `loadPrompt(item)` which sets `recentPrompt` and calls `onSent(prompt)` with that prompt directly.

**Bottom items:**
Help (`<HelpOutlineIcon>`), Activity (`<HistoryIcon>`), and Settings (`<SettingsIcon>`) — visible only when `extended === true`. These are visual elements with no routing or action attached.

**Responsive behavior:**
`@media (max-width: 450px) { .Sidebar { display: none; } }` — the sidebar hides completely on mobile screens.

---

## Styling

**Dark theme:** Body background `#1e1e1e` (near-black). Sidebar background `#2a2a2a`. Main area background inherited from body.

**Gradient logo text:** "GEMINI AI" in the navbar uses `-webkit-linear-gradient(18deg, #ff62fa, #ffd575)` (pink to yellow) with `background-clip: text` and `-webkit-text-fill-color: transparent` — hollow gradient text.

**Greeting gradient:** "Hello, Pawan" uses `-webkit-linear-gradient(16deg, #30c92e, #19d4de)` (green to cyan) with the same clip technique.

**Shimmer loader:** 4 `.load` divs stacked vertically, each 10px tall, full width, with `background: linear-gradient(to right, #ff93d6, #9fe9ff, #de98ff, #ff93d6)` at `background-size: 800px 50px`. The `loader` keyframe shifts `background-position` from `-800px 0` to `800px 0` on a 3s linear loop — producing a sweeping color shimmer across all 4 bars simultaneously.

**Result area scroll:** `.result` has `max-height: 70vh` and `overflow-y: scroll` with `::-webkit-scrollbar { display: none }` — scrollable but scrollbar-hidden.

**Icons:** All icons in `Main.jsx` use Material UI (`@mui/icons-material`) — `AccountCircleIcon`, `SendIcon`, `ImageSearchIcon`, `MicIcon`. Sidebar icons also use MUI — `MenuIcon`, `AddCircleOutlineIcon`, `HelpOutlineIcon`, `HistoryIcon`, `SettingsIcon`, `MessageOutlinedIcon`.

---

## Tech Stack

| Technology | Version | Role |
|---|---|---|
| React | 19.0.0 | Component-based UI, `useContext`, `useState` |
| Vite | 6.2.0 | Dev server, HMR, production build |
| `@google/generative-ai` | 0.24.1 | Official SDK — Gemini 2.0 Flash API calls |
| Material UI (`@mui/icons-material`) | 7.1.0 | All icons in Main and Sidebar |
| `@mui/material` + `@emotion/react/styled` | 7.1.0 | MUI peer dependencies |
| Tailwind CSS | 4.1.3 | Utility classes (used minimally) |
| CSS3 | — | All component-level styling in `Main.css` and `Sidebar.css` |

---

## Project Structure

```
GeminiAiClone/
├── src/
│   ├── components/
│   │   ├── Main/
│   │   │   ├── Main.jsx          # Chat UI — greeting screen, result panel, input bar, send icon logic
│   │   │   └── Main.css          # Gradient text, shimmer loader keyframe, result scroll, responsive breakpoints
│   │   └── Sidebar/
│   │       ├── Sidebar.jsx       # Collapsible sidebar — new chat, recent history, bottom nav icons
│   │       └── Sidebar.css       # Sidebar layout, recent entry hover, mobile hide at 450px
│   ├── conflig/
│   │   └── Gemini.js             # GoogleGenerativeAI init, gemini-2.0-flash model, run() async function
│   ├── context/
│   │   └── Context.jsx           # All state, onSent(), newChat(), delaypara() — shared via React Context
│   ├── assets/
│   │   └── assets.js             # Named exports for all 14 local icon PNG files
│   ├── App.jsx                   # Root — renders Sidebar + Main side by side, wraps with ContextProvider
│   ├── main.jsx                  # React DOM entry point
│   └── index.css                 # Global styles
├── index.html                    # Vite HTML entry
├── vite.config.js                # Vite + React plugin config
└── package.json                  # Dependencies and scripts
```

---

## Getting Started

**Prerequisites:** Node.js 18+ and a Google Gemini API key from [Google AI Studio](https://aistudio.google.com/app/apikey).

**1. Clone the repository**
```bash
git clone https://github.com/tripathipawan/GeminiAiClone.git
cd GeminiAiClone
```

**2. Install dependencies**
```bash
npm install
```

**3. Configure the API key**

Create a `.env` file in the root directory:
```env
VITE_GEMINI_API_KEY=your_gemini_api_key_here
```

**4. Start the development server**
```bash
npm run dev
```

**5. Build for production**
```bash
npm run build
```

---

## Repository

[https://github.com/tripathipawan/GeminiAiClone](https://github.com/tripathipawan/GeminiAiClone)
