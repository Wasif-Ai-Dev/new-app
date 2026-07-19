# Urdu Voice

An Android app that listens for text shared from WhatsApp (or any app that shares plain text), automatically detects whether the text is **English**, **Roman Urdu**, or **Urdu script**, and immediately speaks it aloud in natural Urdu. The translated text is never shown — only audio.

## How the user experiences it

1. In WhatsApp, long-press a message → **Share**.
2. Pick **Urdu Voice**.
3. The app opens as a small "Speaking…" indicator and plays the message in Urdu.
4. When playback ends, the app closes itself.

The main app also has a home screen for pasting text, a searchable history with favorites, and settings for speech rate, pitch, dark mode, and language behavior.

## Tech stack (all free / open source)

| Concern | Choice |
| --- | --- |
| Language | Kotlin |
| UI | Jetpack Compose + Material 3 |
| Architecture | MVVM + Repository |
| DI | Hilt |
| Async | Coroutines + Flow |
| Persistence | Room (history) + DataStore (settings) |
| Language detection | Custom heuristic (Urdu Unicode block + Roman-Urdu lexicon) — 0 KB models, offline |
| Translation | [MyMemory](https://mymemory.translated.net) free API (no key, 5k words/day/IP) |
| Urdu voice | Android system `TextToSpeech` with `ur_PK` (Google Speech Services voice) |
| HTTP | OkHttp |

### Why these free choices

- **MyMemory** — the only reputable translation API with a usable free tier and no signup. Swap it later for offline MT (e.g. bundled ML Kit or a NLLB-200 tiny model) by implementing a second `TranslationService`.
- **System TTS** — every modern Android device has an Urdu voice available through Google Speech Services. If missing, the app prompts the user to install it (no permissions, no downloads).
- If you later want higher quality: **ElevenLabs** or **Google Cloud TTS** (both paid) drop into `UrduTtsEngine` behind the same interface.

## Project structure

```
app/src/main/java/com/urduvoice/app/
├── UrduVoiceApp.kt              # Hilt Application
├── di/                          # Hilt modules (DB, DAOs, context)
├── data/
│   ├── local/                   # Room DB, DAO, Entity, DataStore settings
│   ├── remote/                  # TranslationService (MyMemory)
│   └── repository/              # SpeechRepository (detect → translate → persist)
├── language/                    # LanguageDetector + DetectedLanguage enum
├── tts/                         # UrduTtsEngine wrapping Android TTS
└── ui/
    ├── MainActivity.kt          # Compose host + NavHost
    ├── theme/                   # Material 3 color scheme (dynamic + fallback)
    └── screens/
        ├── home/                # HomeScreen + HomeViewModel
        ├── history/             # HistoryScreen + HistoryViewModel
        ├── settings/            # SettingsScreen + SettingsViewModel
        └── share/               # ShareReceiverActivity (ACTION_SEND + PROCESS_TEXT)
```

## Build

Requirements: Android Studio Koala or newer, JDK 17.

```bash
./gradlew :app:assembleDebug        # debug APK → app/build/outputs/apk/debug/
./gradlew :app:assembleRelease      # unsigned release APK
```

Then install:

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Signing a release APK: create a keystore with `keytool`, add signing config to `app/build.gradle.kts`, and rebuild `:app:assembleRelease`.

## How language detection works

1. Any character in Arabic Unicode blocks → **Urdu** (Urdu script wins mixed content — that's what a listener expects).
2. Otherwise tokens are compared to a curated Roman-Urdu lexicon (~120 unambiguous words: `hai`, `hain`, `kal`, `kidhr`, `main`, `nahi`, `theek` …). If ≥ 25% of tokens or ≥ 2 tokens match → **Roman Urdu**.
3. Otherwise → **English**.

The lexicon deliberately avoids words that collide with English (`is`, `the`) so `"today at 3 pm"` stays English while `"kal 3 pm meeting hai"` becomes Roman Urdu.

## Error handling

Every failure path is surfaced with a plain-language toast or in-screen message:

- No internet / API failure → *"Couldn't play: …"* toast, share activity closes.
- Empty text → *"Please enter some text first"* on home screen.
- Urdu voice not installed → in-app card with instructions to open **Settings → Languages → Text-to-speech output**.
- Translation returned unchanged → treated as failure (would otherwise speak English words with an Urdu voice).

## Privacy

- No analytics, no tracking, no ads.
- Only permission requested: `INTERNET` (needed for translation).
- History is stored locally in Room; nothing leaves the device except the text sent to MyMemory for translation.

## Future work

- Bundle an offline MT model (NLLB-200 distilled or Google ML Kit) and switch based on the *Offline Mode* setting.
- Bundle a neural Urdu voice (e.g. Piper) for higher quality than the OS voice.
- Widget for one-tap "speak clipboard".
- Wear OS companion.
