# Offline Translation — Android Keyboard + Translator (MVP)

A custom Bangla/English keyboard for Android that works **inside** WhatsApp, Messenger,
Telegram, SMS, or any app with a text field — not a replacement for those apps. It adds a
🌐 **Translate** key to the keyboard and a **Translate** option in the text-selection menu,
so a message can be composed in Bangla and sent as English (or vice-versa) without ever
copy-pasting into a separate app, and a received foreign-language message can be translated
in place.

Everything runs **on-device**. The app requests **no `INTERNET` permission at all** (check
`AndroidManifest.xml`), so it is physically incapable of sending typed or translated text
anywhere.

## How to open this project
1. Install Android Studio (Koala/2024.1+ recommended).
2. `File → Open`, select the `OfflineTranslationKeyboard` folder.
3. Let Gradle sync (it will download AndroidX/Compose dependencies — this is a one-time
   build-tooling download, unrelated to translation, which stays offline at runtime).
4. Run the `app` module on a device or emulator (minSdk 24 / Android 7.0+).

## Using it
1. Open the app → **Home** tab → tap **"কীবোর্ড সেটিংসে যান"** to enable the keyboard in
   system settings.
2. Tap **"কীবোর্ড পরিবর্তন করুন"** (or long-press any text field → keyboard icon) and choose
   **Offline Translation**.
3. Open WhatsApp/Messenger/Telegram, type a message in Bangla or English, and tap the green
   🌐 key — the field's text is translated in place, ready to send.
4. To translate a message **you received**: select the text in the chat bubble → tap
   **Translate** in the popup menu that Android shows for any selected text.

## Architecture
```
ime/                  InputMethodService + Compose hosting glue (the keyboard itself)
keyboard/              Key layout data (Bangla vowels/consonants/matras, English QWERTY,
                        symbols, emoji) and the Compose UI that renders them
translation/           TranslationEngine interface + the MVP OfflineTranslationEngine
                        (dictionary/phrase based) + script-based language detector
processtext/           ACTION_PROCESS_TEXT activity — "Translate" in any app's selection menu
data/                  SettingsRepository (SharedPreferences, on-device only)
ui/                    Companion app: Home, Language Models, Keyboard Settings,
                        Translation Settings, Privacy — Jetpack Compose + Navigation
assets/dictionaries/   Bundled bn_en.json / en_bn.json offline dictionaries (shipped in the APK)
```

## Important limitation — please read
The **offline translation engine in this MVP is dictionary/phrase-based, not a neural
machine translation model.** It:
- Matches whole common phrases exactly (e.g. "তুমি কেমন আছো?" → "how are you?").
- Falls back to word-by-word substitution for anything else, leaving unknown words
  (names, slang, rare words) untouched rather than guessing.

This was a deliberate MVP choice: a real neural MT model with useful quality is typically
hundreds of MB to a few GB of weights, and building/training one requires GPU infrastructure
and large parallel corpora that aren't available in this environment. The `TranslationEngine`
interface in `translation/TranslationModels.kt` is intentionally isolated so a future version
can swap in a bundled quantized on-device model (e.g. TensorFlow Lite) — via
`translation/OfflineTranslationEngine.kt` — without changing the keyboard, the app UI, or the
ProcessText activity.

To make the MVP genuinely useful today, extend `app/src/main/assets/dictionaries/bn_en.json`
and `en_bn.json` with more words and phrases relevant to how you actually chat — the format is
plain JSON and needs no code changes to pick up.

## Other known MVP simplifications
- Bangla conjuncts (যুক্তাক্ষর) are typed by combining consonant + হসন্ত + consonant keys
  manually (as specified), rather than auto-detected/auto-ligated.
- Backspace deletes one UTF-16 code unit at a time; most Bangla matras are single code
  points, but a few rare combining sequences may need two backspaces.
- This project has been written and manually reviewed for correctness but has **not** been
  compiled in this environment (no Android SDK / network access here) — please run a Gradle
  sync + build in Android Studio as the first step, and file/fix any small API-level issues
  Android Studio's inspector flags, before treating it as final.

## Adding a new language (Hindi/Arabic/Spanish/French, etc.)
1. Add the language to the `Lang` enum in `translation/TranslationModels.kt`.
2. Add `assets/dictionaries/<src>_<tgt>.json` and `<tgt>_<src>.json` following the existing
   format.
3. Add a matching custom key layout in `keyboard/KeyboardLayouts.kt` if the script needs one
   (Latin-script languages can reuse `EnglishLayout`).
4. List it in `ui/screens/LanguageModelsScreen.kt`.
