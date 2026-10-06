# AML APK Builder — Zip Check Report

**Check ki gayi file:** `files (1).zip` (104 KB, 6 Oct 2026)
**Check kis cheez se kiya:** llama.cpp `master` (commit `1a3011c`, 6 Oct 2026) ke against native C++ ka compile check,
zip integrity, notebook ↔ project consistency, Kotlin/JNI contract, XML/PNG validation.
**Patched package:** `AML_APK_Builder_patched.zip` (isi repo mein)

---

## 1. Verdict (short)

Zip **structurally bilkul theek hai** — koi corrupt entry nahi, koi missing file nahi, notebook ka embedded project
aur alag project zip **100% same** hain (31/31 files, SHA-256 match).

Lekin **ek asli build-breaking issue** mila: default settings (`LLAMA_REF = "master"`) par Colab ka build
**native step par fail** hoga, kyunki llama.cpp ne `llama_model_params::use_mmap` field hata kar
`load_mode` (enum) kar diya hai. Notebook ka auto-fix isko pakad kar purane tag `b6000` par gir jata hai
(wo chal jata hai), lekin pehla build cycle (~kuch minute) zaaya hota hai.

Isi liye ek **patched package** bana diya gaya hai jismein ye fix pehle se hai.

---

## 2. Zip contents & integrity

| Item | Result |
|---|---|
| `unzip -t` (outer + inner) | ✅ no errors |
| Entries | 3 files: `AML_APK_Builder.ipynb` (83 KB), `icon_preview.png` (21 KB), `AML_Android_Project_fixed.zip` (50 KB) |
| Duplicate / unsafe paths (`..`, `/`) | ✅ none |
| Icon PNG | ✅ valid PNG, 884 × 432, 8-bit RGB |
| Notebook JSON | ✅ valid nbformat 4.4, 9 cells, saare Python cells compile hote hain |
| Saare XML (10 files) | ✅ valid XML |
| Project files | 31 files: Kotlin 11, C++ 2, Gradle/config 6, res 11, README/gitignore |

### Notebook ↔ project consistency
Notebook ke cell 3 mein `EMBEDDED_B64` (base64 zip) hai aur alag se `AML_Android_Project_fixed.zip`.
Dono ko decode kar ke compare kiya: **file-by-file SHA-256 bilkul same** (31/31). Matlab "embedded" aur "upload"
dono raaste abhi identical project dete hain — koi drift nahi.

## 3. Code-level checks (jo theek nikle)

- **JNI contract match:** Kotlin `AmlNative.loadModel/generate/stop/unload` ke saare 4 JNI symbols
  `aml_llm.cpp` mein maujood hain, signatures bhi match (jstring/jint/jfloat/jboolean/jobject).
- **Kotlin import graph:** koi unresolved internal import nahi; cross-file references
  (`EngineState.isChatActive`, `AmlTheme`, `ChatViewModel.Factory`, `PromptBuilder`, `formatBytes`, …) sab resolve hote hain.
- **Compose API use:** `LinkAnnotation` / `withLink` / `TextLinkStyles` (Compose 1.7, BOM 2024.12.01 ke saath aate hain),
  `material-icons-extended` ke saare icons, `rememberSaveable`, `mutableIntStateOf` — sab compatible.
- **Versions:** AGP 8.7.3 + Gradle 8.9 + Kotlin 2.0.21 + KSP 2.0.21-1.0.28 + Room 2.6.1 + compileSdk 35 + NDK 27.2.12479018
  (jo notebook install karta hai) — sab ek dusre ke saath consistent.
- **Native code:** `aml_llm.cpp` ke llama.cpp API calls (`llama_model_load_from_file`, `llama_init_from_model`,
  `llama_memory_clear`, `llama_get_memory`, `llama_batch_get_one`, `llama_n_batch`, `llama_vocab_is_eog`,
  `llama_token_to_piece`, sampler chain, `llama_log_set`) — in sab ke naam/signature current llama.cpp mein
  maujood hain (ek field ke ilawa, neeche dekhein). UTF-8/UTF-16 handling sahi hai (partial multibyte buffer hota hai,
  `NewStringUTF` use nahi hota), OOM `java/lang/OutOfMemoryError` mein convert hota hai.
- **No-llama.cpp fallback path** (`AML_HAVE_LLAMA` undefined) bhi compile-clean hai, yaani agar clone fail ho jaye
  to app phir bhi build hoti hai aur `loadModel` false return karta hai — jaisa README claim karta hai.

## 4. 🔴 Issue #1 — `use_mmap` ab llama.cpp mein nahi hai (build fail)

**Kya hua:** llama.cpp ne `llama_model_params::use_mmap` (bool) ko hata kar
`llama_model_params::load_mode` (`enum llama_load_mode`: `AUTO=-1, NONE=0, MMAP=1, MLOCK=2, MMAP_MLOCK=3, DIRECT_IO=4`)
kar diya. Tag-wise verify kiya (sparse clone + `clang -fsyntax-only`):

| llama.cpp revision | date | original `aml_llm.cpp` |
|---|---|---|
| `b6000` | 2025-07-27 | ✅ clean |
| `b10000` | 2026-07-14 | ✅ clean |
| `b10103` | 2026-07-22 | ✅ clean (`use_mmap` maujood) |
| `b10150` | 2026-07-27 | ❌ `no member named 'use_mmap'` (pehla broken tag) |
| `b10450` | 2026-08-15 | ❌ |
| `b11436` | 2026-10-05 | ❌ |
| `master` | 2026-10-06 | ❌ |

**Asar:** notebook cell 1 ka default `LLAMA_REF = "master"` hai → cell 4 (fetch), cell 5 (build) master clone karega →
NDK compile par `aml_llm.cpp:213:12: error: no member named 'use_mmap'` → build FAIL. Phir cell 5 ka auto-fix
(`autofix()`) isi error ko dekh kar `pick_pinned_tag()` chalata hai jo `b6000` pin karta hai — us tag par code clean
compile hota hai, isliye **build doosre attempt mein bach jata hai**. Yaani: result aata hai, magar ek extra
poora build cycle (~kuch minute) aur ek flaky dependency par.

**Fix (patched package mein laga hua):** `setMmapMode()` helper jo SFINAE se dono API shapes support karta hai —
naya code `load_mode = MMAP/NONE` set karta hai, purana code `use_mmap = true/false`. Koi version macro ya `#if`
ki zaroorat nahi. Verify kiya — patched file **clean** compile hoti hai:

`master` ✅ · `b11436` ✅ · `b10450` ✅ · `b10150` ✅ · `b10103` ✅ · `b6000` ✅ · stub-only (llama.cpp ke bina) ✅
(original file master par ❌ — control test.)

Agar aap patched package use nahi karna chahte to sirf `LLAMA_REF` ko `b10103` (ya us se purana, e.g. `b10000`) pin
kar dein — us par purana code bhi theek chalega.

## 5. 🟡 Issue #2 — build-tools 34.0.0 vs compileSdk 35

Cell 2 sirf `build-tools;34.0.0` install karta hai, jabki `compileSdk = 35` hai aur AGP 8.7.3 by default
build-tools `35.0.0` maangta hai. AGP khud auto-download kar leta hai (licenses accept ho chuke hote hain),
isliye ye usually chalta hai — lekin network slow/flaky ho to build ruk sakta hai.
**Patched package mein** cell 2 `build-tools;35.0.0` bhi install karta hai (34.0.0 waise hi rakha gaya hai kyunki
cell 6 ka `aapt2 dump badging` verification usi ka path use karta hai).

## 6. 🟢 Chhote (non-blocking) observations

1. **Gradle wrapper adhoora:** zip mein `gradle/wrapper/gradle-wrapper.properties` hai lekin `gradlew`, `gradlew.bat`
   aur `gradle-wrapper.jar` nahi. Colab ke liye masla nahi (notebook system Gradle 8.9 use karta hai, jo
   properties se match karta hai), magar Android Studio mein project kholte waqt wrapper regenerate karna par sakta hai.
2. **Bogus empty folder:** `app/src/main/res/{values,drawable,mipmap-anydpi,xml}/` — shell brace-expansion ki galti se
   bana hua khali folder (harmless). Patched zip mein hata diya gaya hai.
3. **`local.properties` zip mein nahi** — sahi hai, notebook khud `sdk.dir` likhta hai.
4. **Kotlin/Compose code quality:** `ChatViewModel`, `LlmEngine`, `MarkdownText` waghaira saaf-suthre hain;
   koi TODO/placeholder ya jhoota stub nahi mila. Native code mein JNI local refs delete hote hain, KV cache
   clear hota hai, stop-flag har generate par reset hota hai — sab theek.

## 7. Jo main test nahi kar saka (sandbox ki limit)

- **Poora Gradle/NDK APK build:** is sandbox se `dl.google.com` (Android SDK/NDK) aur Maven Central/Gradle
  distributions **block** hain (GitHub aur PyPI khulte hain). Java bhi install nahi hai. Isliye:
  - Kotlin/Compose ka actual `./gradlew assembleDebug` **nahi chala** — sirf static checks + notebook ka Python code.
  - Native code ka **link stage** test nahi hua — sirf headers ke against **compile/type check** (clang, `-fsyntax-only`),
    jo API-drift pakadne ke liye yahi asli test hai.
  - JNI headers ke liye minimal stub use hua (`jni.h` OpenJDK ka mirror download block tha) — uske naam/signatures
    standard JNI ke mutabiq hain.
- Ye sab Colab par asli run se confirm ho jayega (pehla cell 5 ka build hi final proof hai).

## 8. Patched package — kya badla

`AML_APK_Builder_patched.zip` mein sirf 2 files badli hain (baqi 29 bilkul same):

| File | Change |
|---|---|
| `app/src/main/cpp/aml_llm.cpp` | `setMmapMode()` — purana `use_mmap` aur naya `load_mode` dono support |
| `AML_Android_Project/README.md` | Native build section mein ye compatibility note |
| notebook cell 2 | `build-tools;35.0.0` bhi install |
| notebook cell 3 | `EMBEDDED_B64` refreshed (taake embedded = fixed project, consistency barqarar rahe) |

**use kaise karein:** `AML_APK_Builder_patched.zip` download karein → Colab mein `AML_APK_Builder.ipynb` upload/open
karein → cell 1 mein `SOURCE = "embedded"`, `BUILD_TYPE = "debug"` → Run all. `LLAMA_REF = "master"` ab theek hai
(chahein to `b11436` ya koi bhi tag pin kar lein — dono chalte hain).
