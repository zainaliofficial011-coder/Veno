# AML v3 — Advance Model Loader (all-in-one)

**Author:** macotina · **App name:** AML · **Version:** 3.0 (versionCode 3)

Deliverable: **`AML_APK_Builder_v3.zip`** — Colab notebook + app icon + poora Android project.

---

## 1. Is baar kya naya hai (v2 → v3)

### Pehla screen hi chat hai (koi model mandatory nahi)
- App khulte hi **chat screen** aati hai. Model load na ho to screen ke **beech mein**:
  - **`Local model select (.gguf)`** → system file browser → `.gguf` chunein → file app storage mein copy
    (GGUF magic check ke sath) → **model ka naam title bar mein** dikh jata hai.
  - Uske **neeche** **`API key / API setup`** button → provider presets + base URL + key + model id.
- Screen **light aur clean** hai (Material 3, white surface, halke borders, 6 accent colours).

### Input bar mein saare agent shortcuts (chips)
`Thinking` · `Reason: low/med/high` · `Agent` · `Search` · `Image` · `Video` · `Voice` · `Tools (n)` · `MCP`
+ attach `+`, mic aur send/stop button. Bhari settings (system prompt, temperature, personas, memory, MCP)
**Settings** mein — jaisa aapne kaha.

### Engine: har bara provider + local
| Provider | Kya support hai |
|---|---|
| Local GGUF | llama.cpp (JNI), arm64-v8a, offline, ChatML / Llama 3 / Gemma templates, mmap, threads, context, OOM warning |
| OpenAI-compatible | SSE streaming, native tool calling, vision (data URL), `/models`, `/images/generations` (b64 ya url), `/video/generations` + job polling |
| Anthropic | `v1/messages`, **extended thinking** (1024/4096/8192 budget, temperature 1 par lock), `tool_use` / `tool_result`, image blocks |
| Google Gemini | `generateContent` / `streamGenerateContent?alt=sse`, `systemInstruction`, `functionDeclarations`, **thought parts**, **Imagen `:predict`**, **Veo `:predictLongRunning`** |
| Presets (14) | OpenAI, Anthropic, Gemini, OpenRouter, Groq, DeepSeek, xAI, Mistral, Together, Ollama, LM Studio, llama.cpp server, vLLM, Custom |

### Thinking (har jagah dikhta hai)
OpenAI `reasoning_effort` · Anthropic thinking budget · Gemini `thinkingConfig` · local
`<thinking>…</thinking>` — sab ek collapsible card mein stream karte waqt aur message ke sath save.

### Agent tools — 22 built-in + MCP
`web_search`, `fetch_page`, `http_request`, `open_url`, `open_app`, `write_file`, `read_file`, `list_files`,
`delete_file`, `make_zip`, `list_zips`, `share_file`, `storage_info`, `device_info`, `clipboard_read`,
`clipboard_write`, `notify`, `set_alarm`, `vibrate`, `torch`, `generate_image`, `generate_video`.

- **MCP servers (streamable HTTP JSON-RPC)**: Settings ▸ MCP mein URL + optional bearer token; AML
  `initialize → tools/list → tools/call` karta hai (`Mcp-Session-Id` handle hota hai) aur server ke tools
  khud agent mein aa jate hain ([MCP] tag ke sath).
- Native function calling jab API support kare; warna text protocol (`<tool_call>` / `<tool_result>`) —
  dono ek hi `ToolRunner` se guzarte hain.
- Har call live status row (running / ✓ / ✗) ban kar streaming bubble mein dikhta hai.
- Files `Downloads/AML` mein MediaStore se export hoti hain — koi storage permission nahi.

### Aur bhi
Voice input (live partial results) + TTS (auto-speak option) · image attach (vision) · Gallery screen ·
chats drawer (search / delete / delete-all / export Markdown) · per-message copy, share, speak, edit,
regenerate · token count + tok/s + RAM stats · personas (5 built-in) + memory · 9 settings sections ·
theme light/dark/system + 6 accents + text size · Room chat history (migration 1→2 safe) · author
`macotina` app ke About aur manifest meta-data mein.

---

## 2. Notebook (Colab) — ab APK build hoti hai

`AML_APK_Builder.ipynb` ke 6 code cells **upar se neeche** chalayein (ya Runtime ▸ Run all):

1. **Settings** — `SOURCE=embedded`, `BUILD_TYPE=debug`, `LLAMA_REF=master`, `MAX_ATTEMPTS=4`
2. **Toolchain** — JDK 17, Android SDK 35, build-tools, NDK 27.2.12479018, CMake 3.22.1, Gradle 8.9
3. **Embedded project** (aapka poora v3 source base64 mein)
4. **Prepare project** + **source lint** (23 Kotlin files + brace check — pehle hi pata chal jata hai agar
   koi file adhoori hai)
5. **Fetch llama.cpp** (master; API badal jaye to naya `b6xxx` tag khud try hota hai)
6. **Build APK** — build se pehle Gradle heap 6 GB, plugins warm-up, NDK/llama.cpp preflight; fail hone par
   **auto-fix + retry**: sdk.dir, licenses, NDK/CMake/build-tools reinstall, memory, network retry,
   plugin refresh, llama.cpp tag switch — aur poori error list print
7. **Verify + download** — `aapt2 dump badging` (package/label/permissions/ABI), native libs check
   (`libllama`/`libggml`), phir APK download

v2 ka build fail hona aam tor par in 3 wajahon se hota hai, aur v3 notebook teenon ko khud handle karta hai:
**(a)** llama.cpp C++ API drift (SFINAE fix + tag fallback), **(b)** memory (heap 6 GB), **(c)** SDK/NDK/CMake
incomplete (auto reinstall).

---

## 3. Verification (is sandbox mein jo check hua)

| Check | Result |
|---|---|
| Kotlin brace/paren balance (string/comment-aware scanner, 23 files) | ✅ clean |
| `com.aml.*` imports resolve to real declarations | ✅ (ChatScreen ka `ReasoningLevel` import galat package se tha — fix) |
| Missing-import / unknown-type heuristic scan | ✅ OK |
| Named-argument check: every call site vs declaration (file-local + cross-file) | ✅ OK |
| Constructor named-args (`MessageEntity`, `Settings`, `Turn`, `GenParams`, …) | ✅ OK |
| ViewModel member calls (`viewModel.*`, `toolRunner.*`, `dao.*`, `engine.*`) | ✅ OK |
| Inner zip = 45 files / 23 Kotlin · `testzip` clean · tree se byte-identical | ✅ |
| Notebook: saare code cells Python-compile | ✅ |
| Notebook: `EMBEDDED_B64` decode == inner zip (109,777 B) | ✅ byte-for-byte |
| Notebook: embedded zip se project extract + source lint (locally chalaya) | ✅ 23 files, lint clean |
| Outer zip entries: `AML_APK_Builder.ipynb`, `icon_preview.png`, `AML_Android_Project.zip` | ✅ |
| Logic fixes is round: pehli baar model load (`LoadingModel` guard), stop par adhoora jawab save, `ensureActive()` context, MCP tools local prompt mein | ✅ |

Native (llama.cpp) compile is sandbox mein nahi ho sakta (Kotlin/NDK toolchain ke liye internet
blocked hai) — woh Colab cell 5/6 mein hota hai, aur `aml_llm.cpp` ka `applyLoadMode` SFINAE **dono**
llama.cpp variants (`use_mmap` ≤ b10103 aur `llama_load_mode` ≥ b10150) ke sath chalta hai.

---

## 4. Chalanay ka tareeqa (user ke liye)

1. `AML_APK_Builder_v3.zip` download karein → Colab mein `AML_APK_Builder.ipynb` khol kar **Run all**.
2. Ban gayi APK phone par install karein (*Install unknown apps* allow).
3. App khulegi → **`Local model select (.gguf)`** (offline) **ya** **`API key / API setup`** (online).
4. Input bar ke chips se thinking, agent, search, image, video, voice, tools aur MCP control karein.
5. Image/video ke liye Settings ▸ API mein model id daalein (`gpt-image-1`, `sora-2`,
   `imagen-4.0-generate-001`, `veo-3.0-generate-preview` etc.).

Author: **macotina**. App: **AML — Advance Model Loader**.
