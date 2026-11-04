# whisper-scrawl

A local [whisper.cpp](https://github.com/ggml-org/whisper.cpp) transcriber for
[Scrawl](https://github.com/kitlangton/scrawl)'s `"transcriber": "command"` mode, for languages
Apple's speech engine doesn't cover on your Mac.

## Why

On macOS 26 Scrawl transcribes with Apple's `SpeechTranscriber` whenever it is available. On the
Mac this was written on, `SpeechTranscriber.supportedLocales` lists 45 locales with no Ukrainian,
Russian, Polish, Czech and others, while `DictationTranscriber` lists 54 including them. Scrawl
picks a locale only from the `SpeechTranscriber` list and falls back to `en-US` (`helper/Media.swift`),
so those languages come out as English gibberish. Check your own Mac:

```swift
// swift locales.swift
import Speech
print(await SpeechTranscriber.supportedLocales.map { $0.identifier(.bcp47) }.sorted())
print(await DictationTranscriber.supportedLocales.map { $0.identifier(.bcp47) }.sorted())
```

Scrawl's command mode hands the recording to any local program and reads a JSON transcript back.
This is that program: it runs whisper.cpp on the Mac (Metal on Apple silicon), nothing leaves the machine.

## Requirements

- `brew install whisper.cpp` (provides `/opt/homebrew/bin/whisper-cli`)
- a ggml Whisper model, e.g. `ggml-large-v3-turbo-q8_0.bin` (874 MB) from
  [ggerganov/whisper.cpp](https://huggingface.co/ggerganov/whisper.cpp), SHA-256
  `317eb69c11673c9de1e1f0d459b253999804ec71ac4c23c17ecf5fbe24e259a1`
- the system `/usr/bin/python3` (3.9+, no packages; it doesn't move with Homebrew upgrades)

## Install

```sh
git clone https://github.com/TrueNihilisme/whisper-scrawl.git ~/whisper-scrawl
mkdir -p ~/.local/share/whisper-models
curl -L -o ~/.local/share/whisper-models/ggml-large-v3-turbo-q8_0.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-turbo-q8_0.bin
shasum -a 256 ~/.local/share/whisper-models/ggml-large-v3-turbo-q8_0.bin
```

Then add to `~/.config/scrawl/config.json` (keep the keys already there). Scrawl runs the argv
without a shell, so every path must be absolute: replace `/Users/you`.

```json
"transcriber": "command",
"transcribeCommand": [
  "/Users/you/whisper-scrawl/whisper-scrawl",
  "--model", "/Users/you/.local/share/whisper-models/ggml-large-v3-turbo-q8_0.bin",
  "--language", "uk",
  "{audio}"
]
```

`--language` is any Whisper language code (`uk`, `ru`, `pl`, `cs`, …) or `auto`.
Settings › Recording › Transcription should now show **Command**.

Try it by hand on a 16 kHz mono WAV:

```sh
./whisper-scrawl --model ~/.local/share/whisper-models/ggml-large-v3-turbo-q8_0.bin --language uk clip.wav
```

## How it works

Runs `whisper-cli -ojf` in its own temporary directory (Scrawl may run two transcriptions at once),
joins tokens into words (a token starting with a space starts a new word) and prints one JSON object
in Scrawl's format (`"version": 1`, `text`, `segments`, `words`). `estimated` is `true`: without DTW,
whisper.cpp token times are the decoder's estimate, not an aligner's measurement, so Scrawl won't cut
keyframes on spoken cues.

Tested with whisper.cpp 1.9.5 and Scrawl 0.3.13 on an M4: a 6.7 s Ukrainian clip was transcribed word
for word in about 16 s including model load; an empty environment (`env -i`) and two parallel runs work
and leave no temporary files; a real Scrawl capture reports `transcript.source = command:whisper-scrawl`.

## Undo

Remove the two keys from `config.json`, delete the clone and the model, and `brew uninstall whisper.cpp`
if nothing else uses it.

## License

MIT
