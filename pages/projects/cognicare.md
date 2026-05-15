COGNICARE — EMBEDDED PATIENT MONITORING ASSISTANT
Status: Complete · Stack: Python, Raspberry Pi 4B, GPIO C, Ollama, PyAudio, SoX, pyttsx3 · Course: ECE 198 Project Studio · Source: GitHub

THE PROBLEM
Hospital-Induced Delirium is common, particularly in elderly patients, and its primary risk factor is isolation. The fix is not complicated, lonliness is easily preventable and a major cause.
CogniCare is a bedside assistant that provides that company, monitors for signs of distress or confusion, and flags when a medical professional might be needed, helping reduce nurse workload.

WHAT IT DOES
CogniCare listens. It wakes on a name, holds a conversation, and goes back to waiting when you're done. The conversation runs through a locally-hosted 20B parameter model via Ollama — entirely on the Pi, no internet required for inference.
Every audio clip is cleaned through SoX before anything is analysed to increase accuracy. RMS volume and dominant frequency are extracted from each clip. Any parameter if above/below threshold its threshold triggers a physical buzzer alert via GPIO — three pulses on pin 18.
SpO2 and temperature sensors are integrated for periodic vitals checks, with prompts delivered through the conversation. An LCD provides simple expressive feedback to make the interaction feel less like talking to a device. The latency is ~250ms on a lightweight pi.

WHAT I LEARNED
The speech analysis — pitch, amplitude, silence — was a deliberate choice for constrained hardware, but it's accuracy can be improved with ml instead of min/max threshold values.
The Grove sensors given that did not have Python3 libraries. So we wrote the bindings ourselves, which was not in the project plan and took longer than most of what was. 
The sensors are consumer-grade, not medical. The readings are useful as a proof of concept and the architecture is straightforward to upgrade.

RECEPTION
"I'm going to the hospital tomorrow for a hip replacement, it would be cool if this were available."
— LinkedIn commenter.
