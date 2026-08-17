# Home-Assistant-Voice-PE-Wake-Word-Hey-Alfred-
Wake Word "Hey Alfred" for Home Assistant Voice PE 

Hey Alfred — Custom microWakeWord Model

A custom on-device wake word model for Home Assistant Voice Preview Edition and other ESPHome-based satellites, trained to detect the phrase "Hey Alfred" using the microWakeWord framework.

Files
File	Size	Description
hey_alfred.tflite	135 KB	Quantized streaming inference model (runs on-device on the ESP32-S3)
hey_alfred.json	2 KB	Model metadata (detection thresholds, sliding window size, calibration)

These sizes are normal — microWakeWord models are intentionally small so they can run continuously on the ESP32-S3's limited RAM/compute without needing to stream audio to a server.

Training Method

Trained locally (no cloud GPU service) using TaterTotterson/microWakeWord-Trainer-Nvidia-Docker, a Docker-based training UI ("Wake Word Studio") that runs the full microWakeWord pipeline on a local Nvidia GPU.

Hardware:

GPU: Nvidia RTX 4070 (12GB VRAM)
CPU: Intel Core i7-9700K (8 cores)
RAM: 32GB
All other GPU workloads (LLM inference, STT/TTS containers) were stopped for the duration of the run to dedicate full hardware to training.

Configuration:

Wake phrase: hey alfred
Language: English (en)
Accent emphasis: Mixed English
TTS source: Four-provider ensemble (OmniVoice, Qwen3, MOSS, Piper) — chosen for broader voice/accent diversity to reduce false triggers and improve generalization
Personal samples: 11 real recorded takes of "hey alfred" (own voice) included alongside synthetic samples
Samples generated: 50,000 (100/batch), passed through an automated quality QA gate that rejected malformed TTS output (clipped audio, silence, static, overly long/rambling generations, etc.) before augmentation
Negative/background datasets: MIT environmental RIRs (room impulse responses), AudioSet, FMA (Free Music Archive), CHiME-Home
Augmentation: room simulation + background noise mixing applied to all 50,000 samples
Training steps: 40,000

Detection calibration (auto-selected):

Cutoff: 0.97
Sliding window: 5
Recall: 95.52%
Ambient false-accept rate: 0.207 per hour (~1 false trigger per ~5 hours of ambient background noise)
Time Taken
Stage	Elapsed time
Sample generation (50,000 samples)	1 day, 15h 34m
Augmentation (50,000 samples)	1h 24m
Training (40,000 steps)	2h 40m
Total	1 day, 19h 39m

Notes on the timeline:

Sample generation was by far the largest time cost, not the actual training step. This is a direct consequence of prioritizing quality (large sample count, four-provider TTS ensemble, real-world background datasets) over speed.
One TTS provider (MOSS) required a second generation pass after its QA gate rejected ~33% of its first-pass output (mostly "too long or rambling" generations).
The run hit one interruption: chime_home.tar.gz was downloaded corrupted (gzip/tar EOF error) partway through, likely from a network hiccup, and had to be deleted and re-downloaded before training could proceed past that stage. This cost some additional wall-clock time but was unrelated to the GPU/training pipeline itself.
A separate, unrelated crash (exit_code: 999) occurred earlier in the run due to the host machine running low on disk space (training datasets alone reached ~87.66 GB). Freeing disk space (~200GB+ available) resolved it.
Known Limitations
This model has not yet been validated against the built-in Home Assistant Voice PE hardware in real-world conditions — accuracy figures above are from the trainer's own streaming inference test against ambient audio tracks, not live device testing.
No manually-labeled negative samples (e.g. "Alfred" spoken without "hey") were included in this training run — the trainer UI's manual sample import had no working path to classify uploads as negatives (they always imported as positive samples), so this refinement was dropped for this run. A future training pass could add this via the Captured Audio review flow once real false-wake clips are available from the device.
Deployment (Home Assistant Voice PE / ESPHome)

Add the model to your Voice PE's ESPHome YAML under micro_wake_word:

yaml
micro_wake_word:
  id: mww
  models:
    - model: https://github.com/drewballz1/Home-Assistant-Voice-PE-Wake-Word-Hey-Alfred-/raw/refs/heads/main/hey_alfred.json
      id: hey_alfred

After flashing, select "Hey Alfred" from the wake word dropdown in Settings → Devices & Services → ESPHome on the device's page.
