# Local LLM Voice Chatbot

A voice-controlled chatbot that combines **NVIDIA Riva** for speech recognition and speech synthesis with **Ollama** running Llama 3. The Python client runs on a Windows host and connects to Riva and Ollama services hosted in an Ubuntu virtual machine with GPU passthrough.

The goal is to speak a question into a microphone, send the recognized text to a locally hosted language model, and hear the response spoken aloud.

## How it works

1. **Speech recognition (ASR):** NVIDIA Riva transcribes microphone audio.
2. **Language model:** The Python client sends the transcript to Llama 3 through Ollama.
3. **Speech synthesis (TTS):** NVIDIA Riva converts the generated response to audio and plays it through the selected output device.

```text
Windows host
Microphone -> local_chatbot.py -> speakers
                    |     ^
                    v     |
          Ubuntu VM (Hyper-V)
          NVIDIA Riva ASR/TTS
          Ollama + Llama 3
```

## Requirements

- Windows host with Python installed.
- Hyper-V and an Ubuntu 22.04 VM.
- GPU passthrough configured for the VM. Follow the [GPU passthrough guide](https://github.com/puichuentam/dxgkrnl_ubuntu) for the VM setup steps.
- Docker and NVIDIA Riva Quickstart 2.15.1 configured inside the VM.
- Ollama and a Llama 3 model installed inside the VM.
- A working microphone and audio output device on the Windows host.
- Network connectivity from the Windows host to the VM on the Riva and Ollama ports (50051 and 11434 by default in this setup).

## Setup

### 1. Set up the Ubuntu VM

Create an Ubuntu 22.04 VM in Hyper-V, enable Enhanced Session Mode, and configure XRDP/PulseAudio as needed for your environment. Complete the GPU passthrough setup using the guide linked above.

### 2. Install NVIDIA Riva in the VM

1. Log in to the NVIDIA NGC Catalog, install the NGC CLI, and create an API key.
2. Configure the CLI and download Riva Quickstart:

   ```bash
   ngc config set
   ngc registry resource download-version nvidia/riva/riva_quickstart:2.15.1
   ```

3. Enter the downloaded `riva_quickstart_v2.15.1` directory and initialize Riva:

   ```bash
   cd riva_quickstart_v2.15.1
   bash riva_init.sh
   ```

   Initialization can take around 15–20 minutes, depending on the environment.

4. Increase the configured TTS preprocessor length limits in the `riva-model-repo` Docker volume. Start a temporary Ubuntu container with the volume mounted:

   ```bash
   docker run -it -v riva-model-repo:/data ubuntu
   apt-get update
   apt-get install -y nano
   cd /data/models/tts_preprocessor-English-US
   nano config.pbtxt
   ```

   In `config.pbtxt`, set both values to `4096`:

   ```text
   max_input_length: 4096
   max_sequence_length: 4096
   ```

   Save the file and exit the container. Then start Riva from the Riva Quickstart directory (or the directory containing `riva_start.sh`, according to your installation):

   ```bash
   bash riva_start.sh
   ```

   Riva listens on port `50051` by default in this setup. Keep the terminal running so you can monitor its logs.

### 3. Install and start Ollama in the VM

Install Ollama and download a Llama 3 model:

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3:instruct
```

Find the VM's IP address:

```bash
ip addr
```

Start Ollama listening on the VM's network address:

```bash
OLLAMA_HOST=<VM_IP>:11434 ollama serve
```

Replace `<VM_IP>` with the VM's actual IP address. Keep this terminal running as well.

**Model-name check:** This guide's download command uses `llama3:instruct`, but the current Python script requests `model='llama3'`. Check the installed model names with `ollama list` and make the tag in `local_chatbot.py` match the model you downloaded.

### 4. Configure the Python client on Windows

Download or clone this repository and open Command Prompt or PowerShell in the project directory. Install the Python dependencies:

```powershell
python -m pip install nvidia-riva-client requests ollama
```

In `local_chatbot.py`, update the VM IP address in both locations where it is hard-coded:

- `ArgsASR.server` — the Riva endpoint, normally port `50051`.
- `Client(host=...)` — the Ollama endpoint, normally port `11434`.

The script currently contains an example IP address; replace it with the IP address of your own VM. Ensure Windows can reach both ports through the VM and firewall configuration.

### 5. Select audio devices and run the chatbot

List available microphone devices:

```powershell
python .\local_chatbot.py --list-input-devices
```

List available audio output devices:

```powershell
python .\local_chatbot.py --list-output-devices
```

Use the device indices shown on your own machine. For example, if your microphone index is `1` and your output index is `3`:

```powershell
python .\local_chatbot.py --input-device 1 --output-device 3
```

The script has different default device indices, so passing the correct indices explicitly is recommended.

## Using the chatbot

1. Start Riva and Ollama in the Ubuntu VM.
2. Start `local_chatbot.py` on Windows with your chosen audio device indices.
3. Say **“Hello”** to activate the conversation. The chatbot should respond, “Here!”
4. Ask a question and wait for the spoken response. The setup guide reports roughly 20–30 seconds for a response on its test system; performance depends on your hardware and model.
5. Say **“Stop”** to end the active conversation, or press `Ctrl+C` in the terminal to stop the Python program.

The current wake-word checks are case-sensitive, so use lowercase `hello` and `stop` if recognition does not trigger as expected.

## Configuration notes and current limitations

- **VM IP addresses are hard-coded** in the Python file and must be updated for your network.
- **Audio device indices vary by machine.** Use the device-listing options rather than copying the example indices.
- **Model tags must match.** The Ollama model name in the `url.chat(...)` call must exist in the VM's Ollama installation.
- **Language settings are currently `en-US`.** The ASR and TTS settings are configured for English (US).
- **Riva and Ollama must remain running** in the VM while the Windows client is in use.
- The setup requires GPU passthrough and downloads NVIDIA Riva assets through NGC; it is not a single-command install.

## Stopping the services

Press `Ctrl+C` in the Windows terminal to stop the Python client. Stop the Riva and Ollama processes in their respective VM terminals when finished.

## License

No license is specified yet. Add a license before publishing if you want others to know how they may use, modify, and redistribute this project.
