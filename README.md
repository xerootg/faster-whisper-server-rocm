# This is a Hack
I took the build for rocm/CTranslate2 and grafted it into the dockerfile for faster-wisper-server, but significantly downgraded. py3.10 instead of 3.11 and faster-whisper 0.10 instead of 1.0.3. this means fewer models (no distill?) and other compromises. BUT it does work. I am building gfx1100 (I've got a RX 7600), you'll need to set your correct version in the Dockerfile if you need a different target. The only place seems like CTranslate2's cmake setup.

# Faster Whisper Server
`faster-whisper-server` is an OpenAI API compatible transcription server which uses [faster-whisper](https://github.com/SYSTRAN/faster-whisper) as it's backend.
Features:
- GPU and CPU support.
- Easily deployable using Docker.
- Configurable through environment variables (see [config.py](./faster_whisper_server/config.py)).
- OpenAI API compatible.

Please create an issue if you find a bug, have a question, or a feature suggestion.

## OpenAI API Compatibility ++
See [OpenAI API reference](https://platform.openai.com/docs/api-reference/audio) for more information.
- Audio file transcription via `POST /v1/audio/transcriptions` endpoint.
    - Unlike OpenAI's API, `faster-whisper-server` also supports streaming transcriptions(and translations). This is usefull for when you want to process large audio files would rather receive the transcription in chunks as they are processed rather than waiting for the whole file to be transcribe. It works in the similar way to chat messages are being when chatting with LLMs.
- Audio file translation via `POST /v1/audio/translations` endpoint.
- (WIP) Live audio transcription via `WS /v1/audio/transcriptions` endpoint.
    - LocalAgreement2 ([paper](https://aclanthology.org/2023.ijcnlp-demo.3.pdf) | [original implementation](https://github.com/ufal/whisper_streaming)) algorithm is used for live transcription.
    - Only transcription of single channel, 16000 sample rate, raw, 16-bit little-endian audio is supported.

## Quick Start
Using Docker
```bash
docker run --gpus=all --publish 8000:8000 --volume ~/.cache/huggingface:/root/.cache/huggingface fedirz/faster-whisper-server:cuda
# or
docker run --publish 8000:8000 --volume ~/.cache/huggingface:/root/.cache/huggingface fedirz/faster-whisper-server:cpu
```
Using Docker Compose
```bash
curl -sO https://raw.githubusercontent.com/fedirz/faster-whisper-server/master/compose.yaml
docker compose up --detach faster-whisper-server-cuda
# or
docker compose up --detach faster-whisper-server-cpu
```
## Usage
### OpenAI API CLI
```bash
export OPENAI_API_KEY="cant-be-empty"
export OPENAI_BASE_URL=http://localhost:8000/v1/
```
```bash
openai api audio.transcriptions.create -m distil-medium.en -f audio.wav --response-format text

openai api audio.translations.create -m distil-medium.en -f audio.wav --response-format verbose_json
```
### OpenAI API Python SDK
```python
from openai import OpenAI

client = OpenAI(api_key="cant-be-empty", base_url="http://localhost:8000/v1/")

audio_file = open("audio.wav", "rb")
transcript = client.audio.transcriptions.create(
    model="distil-medium.en", file=audio_file
)
print(transcript.text)
```

### CURL
```bash
# If `model` isn't specified, the default model is used
curl http://localhost:8000/v1/audio/transcriptions -F "file=@audio.wav"
curl http://localhost:8000/v1/audio/transcriptions -F "file=@audio.mp3"
curl http://localhost:8000/v1/audio/transcriptions -F "file=@audio.wav" -F "streaming=true"
curl http://localhost:8000/v1/audio/transcriptions -F "file=@audio.wav" -F "streaming=true" -F "model=distil-large-v3"
# It's recommended that you always specify the language as that will reduce the transcription time
curl http://localhost:8000/v1/audio/transcriptions -F "file=@audio.wav" -F "streaming=true" -F "model=distil-large-v3" -F "language=en"

curl http://localhost:8000/v1/audio/translations -F "file=@audio.wav"
```

### Live Transcription
[websocat](https://github.com/vi/websocat?tab=readme-ov-file#installation) installation is required.
Live transcribing audio data from a microphone.
```bash
ffmpeg -loglevel quiet -f alsa -i default -ac 1 -ar 16000 -f s16le - | websocat --binary ws://localhost:8000/v1/audio/transcriptions
```
Streaming audio data from a file.
```bash
ffmpeg -loglevel quiet -f alsa -i default -ac 1 -ar 16000 -f s16le - > audio.raw
# send all data at once
cat audio.raw | websocat --no-close --binary ws://localhost:8000/v1/audio/transcriptions
# Output: {"text":"One,"}{"text":"One,  two,  three,  four,  five."}{"text":"One,  two,  three,  four,  five."}%
# streaming 16000 samples per second. each sample is 2 bytes
cat audio.raw | pv -qL 32000 | websocat --no-close --binary ws://localhost:8000/v1/audio/transcriptions
# Output: {"text":"One,"}{"text":"One,  two,"}{"text":"One,  two,  three,"}{"text":"One,  two,  three,  four,  five."}{"text":"One,  two,  three,  four,  five.  one."}%
```
