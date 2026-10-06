# AML v2 — Chat screen + API + Agent + Media upgrade

**Naya package:** `AML_APK_Builder_v2.zip` (notebook + fixed project + aapka icon)
**Purana:** `AML_APK_Builder_patched.zip` (v1, sirf llama.cpp `use_mmap` fix) — ye v2 us se aage hai.

---

## 1. Kya kya add hua (aapki list ke hisaab se)

| Aapki request | Status | Kahan |
|---|---|---|
| Chat screen update + best UI | ✅ | `ui/ChatScreen.kt` (naya: thinking card, tool cards, media bubbles, gallery, quick chips, backend strip) |
| API support (base URL, API key, model id) | ✅ | `engine/ApiEngine.kt` + Settings screen |
| Local model run (GGUF) | ✅ | wahi llama.cpp JNI, ab Settings se ctx/threads/mmap/template |
| GGUF model run | ✅ | Local tab: import + Load + prompt-template chips |
| Image model run | ✅ | Image tab + `generate_image` tool → `/images/generations` |
| Video model run | ✅ | Video tab + `generate_video` tool → `/video/generations` (job polling) |
| Model ko har cheez ka access (agent) | ✅ | 13 tools: search, fetch, files, zip, storage, device, image, video, open URL |
| Search kare | ✅ | `web_search` (DuckDuckGo, key ki zaroorat nahi) + `fetch_page` |
| File banaye | ✅ | `write_file` / `read_file` / `list_files` / `delete_file` (workspace + Downloads/AML export) |
| Zip banaye | ✅ | `make_zip` / `list_zips` |
| Storage access | ✅ | `storage_info`, MediaStore export, workspace + media folders, Gallery |
| Thinking | ✅ | `<thinking>` + `reasoning_content` → collapsible "Thinking" card, saved with message |
| Vision (image bhejna) | ✅ | chat mein image attach → `image_url` data URL (API model) |
| App name AML + icon | ✅ | `strings.xml` = AML, adaptive icon = white bg + blue AML wordmark (aapka `icon_preview.png`) |

## 2. Backend routing

| Setting | Chat | Vision | Tools | Image/Video |
|---|---|---|---|---|
| `Local GGUF` | llama.cpp (offline) | ❌ (text-only, app batata hai) | ✅ text-protocol | ❌ (API chahiye) |
| `API` | `/chat/completions` (SSE streaming) | ✅ | ✅ native `tools` ya text-protocol | ✅ |

Base URL koi bhi chalta hai: `https://api.openai.com/v1`, `http://192.168.1.20:8080/v1` (Ollama / LM Studio /
llama.cpp server), OpenRouter, Groq, DeepSeek, Together, vLLM. Plain HTTP jaan-boojh kar allowed hai
(`network_security_config.xml`) — LAN servers ke liye.

## 3. Agent tools (13)

`web_search` · `fetch_page` · `open_url` · `write_file` · `read_file` · `list_files` · `delete_file` ·
`make_zip` · `list_zips` · `storage_info` · `device_info` · `generate_image` · `generate_video`

- **API + native tool calling ON** → `tools` array bhejta hai, streaming `tool_calls` assemble karta hai.
- **Local model / native tools OFF** → text protocol: model `<tool_call>{...}</tool_call>` likhta hai,
  AML `<tool_result>…</tool_result>` wapas bhejta hai. Isliye har model ke saath chalta hai.
- Max tool rounds aur result truncation Settings se control hote hain.
- Files: `files/workspace`, media: `files/media`, export: `Downloads/AML` (MediaStore, **koi permission nahi**).
  Manifest mein sirf `INTERNET` + `ACCESS_NETWORK_STATE` hain.

## 4. Testing jo main kar saka (aur jo nahi)

Is sandbox mein Java/Android SDK nahi (aur `dl.google.com`, Maven Central, Gradle distributions block hain),
isliye **Gradle build yahan nahi chala**. Jo verify hua:

| Check | Result |
|---|---|
| llama.cpp C++ (`aml_llm.cpp`) vs **master**, `b11436`, `b10450`, `b10150`, `b10103`, `b6000` | ✅ clean compile (clang `-fsyntax-only`) |
| Kotlin: brace/paren balance, saari files | ✅ balanced |
| Saare `com.aml.*` imports resolve | ✅ none missing |
| Naye 59 symbols declared + cross-file refs | ✅ 59/59 |
| Compose imports (`Icons.Filled.*`, layout, unit, Alignment…) | ✅ koi missing nahi |
| Duplicate top-level declarations (same package) | ✅ koi nahi |
| Notebook: JSON valid, Python cells compile, `EMBEDDED_B64` == project zip | ✅ |
| XML valid (manifest, themes, `file_paths`, `network_security_config`) | ✅ |
| Room migration v1→v2 (`kind`, `mediaPath`, `thinking`) | likha gaya, purane chats safe |

**Jo abhi bhi Colab par hi test hoga:** `assembleDebug` (Kotlin compile + link), aur actual API calls.
Notebook cell 5 khud errors print karta hai aur retry/auto-fix karta hai.

## 5. Pehli baar chalane ka tareeqa

1. `AML_APK_Builder_v2.zip` download → Colab mein `AML_APK_Builder.ipynb` open → **Run all**.
2. APK install → app khulega **model selector** par:
   - **Local tab** → `Import .gguf` → **Load** (offline chat), ya
   - **API tab** → base URL + API key + model id → **Test connection** → **Use this API**.
3. **Settings (⚙)** → image model id, video model id, thinking, tools, temperature, max tokens, system prompt.
4. Chat mein top bar ke 🧠 (thinking) aur 🔧 (tools) se turant on/off; drawer se Settings/Gallery/New chat.

## 6. Chhote fix jo isi version mein hain

- **v1 ka `use_mmap` bug** (llama.cpp `b10150`+ ne field rename ki) — v2 mein bhi fix maujood hai (SFINAE helper).
- Notebook ab `build-tools;35.0.0` bhi install karta hai (compileSdk 35 / AGP 8.7 se match).
- Zip mein se woh ghalat khali folder `res/{values,drawable,mipmap-anydpi,xml}/` nikal diya gaya.

## 7. Limits (jaan-boojh kar)

- Local GGUF = text-only (vision ke liye API model).
- Video endpoint provider-wise different hai; AML dono shapes handle karta hai (direct asset ya job id + polling
  ~10 min), warna saaf error deta hai.
- Message edit/regenerate abhi nahi; schema badalne par Room version bump + Migration likhna hoga (v2 ka example maujood hai).
