# Build Your Own Tiny AI Companion Robot
### A voice-controlled tracked robot with moving eyes, grippers, and a brain powered by a large language model — all for under $140.

---

## Introduction

There's something deeply satisfying about a robot that looks back at you. Not a screen with a face drawn on it, but a physical machine with two eyeballs that track your movement, a pair of small clawed hands that open and close when it gets excited, and a voice that actually understands what you say and responds intelligently.

That's exactly what you're going to build in this project.

The robot rides on rubber tracks, so it handles carpet and desk surfaces without complaint. It listens through a microphone, thinks through a large language model (either running in the cloud or locally on the board), and speaks its responses aloud through a small speaker. When it answers you, its eyes move to match the emotion of what it's saying. When it wants to wave hello, it does. When you ask it what it can see, it takes a photo and tells you.

The whole thing is built around a Raspberry Pi Zero 2 W — a board the size of a stick of gum that costs $15. Total parts cost lands between $80 and $140 depending on where you source them. You don't need to be an electronics engineer to build this. If you can solder a header pin and run a Python script, you can finish this project in a weekend.

---

## What You'll Need

### Parts List

| Part | Specification | Where to Buy | Approx. Cost |
|------|--------------|-------------|-------------|
| Raspberry Pi Zero 2 W | 1 GHz quad-core, 512 MB RAM, built-in Wi-Fi | raspberrypi.com, Adafruit, PiShop | ~$15 |
| Micro SD card | 32–64 GB, Class 10 or faster | Amazon, any electronics store | $5–10 |
| Tracked tank chassis kit | 2WD DC motors, rubber tracks, ~15–18 cm long | Freenove (~$45) or generic AliExpress (~$25) | $25–50 |
| Motor driver breakout | DRV8833 dual H-bridge (preferred) or L298N | Adafruit, Amazon | $3–6 |
| Raspberry Pi Camera Module | v2 8MP or v3 12MP, CSI ribbon connector | raspberrypi.com, Amazon | $20–25 |
| Micro servos | 4× SG90 or MG90S (for the eyes) | Amazon, AliExpress | $8–16 |
| Robot gripper claws | 2× micro servo-based claws | Amazon, AliExpress | $10–20 |
| USB microphone | Any small USB mic, or I2S MEMS module | Amazon | $5–10 |
| Speaker + amp | 3W speaker + MAX98357 I2S amp breakout | Adafruit, Amazon | $5–10 |
| LiPo battery + boost converter | 3.7V 1000–2000 mAh LiPo + 5V boost to 3A | Adafruit, Amazon | $8–12 |
| Miscellaneous | Jumper wires, M2.5 standoffs, hot glue, header pins, kapton tape | — | ~$10 |

**Total: $104–184, realistic mid-range ~$120–140**

### Tools Required

- Soldering iron and solder
- Wire strippers and small snips
- Small Phillips head screwdriver
- Hot glue gun
- A second computer (to flash the SD card and SSH into the Pi)
- Multimeter (optional but very useful for checking wiring)

### Software You'll Install on the Pi

- Raspberry Pi OS Lite (64-bit, Bookworm)
- Python 3.11+
- `picamera2`, `gpiozero`, `speech_recognition`, `gTTS`, `playsound`
- Either Ollama (for local LLM) or an API key for OpenAI, Anthropic Claude, or Grok

---

## How It Works

Before you start soldering, it helps to understand the loop that makes the robot come alive.

When you press the push-to-talk button (or trigger a wake word), the microphone records your voice. That audio is passed through a speech-to-text engine and converted into a text string. That string is sent to a large language model along with a system prompt that tells the LLM it's a small tracked robot with eyes, hands, and a camera. The LLM responds with a JSON object containing two things: the text it wants to speak, and a set of physical actions — where to move the eyes, whether to open or close the grippers, and whether to drive the tracks forward or back. Your Python script parses that JSON, speaks the text through the speaker, and simultaneously drives the servos and motors to animate the response. The whole cycle takes two to five seconds on a cloud LLM, or ten to thirty seconds if you run a small model locally on the Pi.

The camera is used on demand. When you ask the robot what it can see, the script captures a still frame, encodes it to base64, and sends it to a vision-capable LLM endpoint. The LLM describes what's in the image and the robot speaks the description aloud.

---

## Part One: Building the Chassis

### Step 1 — Assemble the Tracked Chassis

Open your tank chassis kit and lay all the parts out on your workbench before you start. Most kits include two aluminium or acrylic side plates, two DC geared motors with shaft collars, rubber track loops, a set of idle wheels, and a bag of screws.

Press each motor into its mounting bracket and secure with the provided screws. The motor shafts should point outward, away from the centre of the chassis. Slide the drive sprocket wheels onto each shaft — these are the toothed wheels that grip the inside of the rubber tracks. Add the idle wheels along the length of each side plate; these are the smooth rollers that keep the track tensioned. Once all wheels are mounted, loop the rubber track around the drive sprocket and all the idle wheels on each side. It takes a little force to stretch the track into place — this is normal.

Flip the chassis over and attach the bottom plate if your kit includes one. Your chassis should now sit flat on the table and roll freely when you push it.

**Test it now:** Push the chassis across your desk by hand. Both tracks should roll smoothly with no binding or scraping. If a track pops off, the idle wheel spacing may need adjusting — loosen the mounting screws slightly and shift the wheel position until the track sits evenly.

### Step 2 — Mount the Pi Zero 2 W

Solder the 40-pin GPIO header onto your Raspberry Pi Zero 2 W if it didn't come pre-soldered. Place the Pi on the top plate of the chassis and mark the four mounting hole positions with a pen. Drill or punch these holes if your chassis plate doesn't already have them, then mount the Pi using M2.5 nylon standoffs — use nylon, not metal, to prevent the chassis body from shorting against the Pi's GPIO pads.

Nylon standoffs should raise the Pi about 5–8mm above the chassis plate. This leaves enough clearance for the motor driver board you'll add in the next step.

### Step 3 — Mount the Motor Driver

The DRV8833 breakout board is tiny — roughly the size of a postage stamp. You can mount it directly on the chassis plate beside the Pi using a small piece of double-sided foam tape or a single M2 screw through its mounting hole. The L298N is larger and may need to go on the underside of the top plate or on a small riser bracket.

You'll wire the motor driver properly in Part Three, after the OS is running. For now, just get it physically mounted.

---

## Part Two: Building the Head (Eyes + Camera)

This is the most creative part of the build and the piece that makes the robot look alive. Take your time here.

### Step 4 — Print or Source the Eye Mechanism

Each eye needs two servos: one for horizontal pan (looking left and right) and one for vertical tilt (looking up and down). That's four servos total for both eyes.

If you have access to a 3D printer, search Thingiverse for "animatronic eye mechanism SG90" — there are dozens of free designs sized for SG90 servos. Print two eye assemblies. Standard PLA is fine. Print time is roughly two to four hours per eye at standard quality.

If you don't have a 3D printer, you can build a simple version with foam craft balls (about 35–40mm diameter), two small squares of acrylic or thick cardboard as servo mounts, and some stiff wire to link the servo horn to the eyeball. Hot glue holds everything together adequately for a desk robot that isn't going to take any impacts.

Drill or carve a flat on each foam ball and glue a printed pupil (just print a black circle on paper and cut it out) onto the flat. The servo horn should connect to a short arm that cups or grips the back of the eyeball — when the servo rotates, the eyeball pans with it.

### Step 5 — Build the Head Bracket

The head bracket holds both eye assemblies and the camera in a fixed spatial relationship. Cut a small rectangle of 3mm acrylic or plywood, approximately 80mm wide and 50mm tall. This will be the head's faceplate.

Position the two eye assemblies side by side with roughly 30–40mm between their centres — similar to human eye spacing. The camera module mounts centrally between and slightly below the eyes, so the robot's "gaze direction" and its actual field of view are naturally aligned.

Hot glue or screw the pan servos for each eye flush to the back of the faceplate. The tilt servos mount perpendicular to the pan servos — rotating the eyeball up and down. Take your time getting the eye centred in its socket; misalignment here is very obvious once the robot is running.

Mount the camera module by threading the CSI ribbon cable through a small slot in the faceplate, then securing the camera with a tiny dab of hot glue or a 3D-printed clip. Do not put any mechanical stress on the CSI connector — it's fragile.

### Step 6 — Attach the Head to the Chassis

The head bracket should mount on the front face or front top edge of the chassis, raised enough that the camera has a clear forward view and isn't looking at the top of the chassis plate. A short pair of acrylic standoffs or even a small wooden block works well. The goal is to have the eyes approximately at "face height" when the robot is on a desk — about 60–80mm above the surface.

---

## Part Three: Hands and Wiring

### Step 7 — Mount the Gripper Claws

The servo-based gripper claws mount on each side of the chassis, roughly midway along the body. They don't need to reach anything — they're primarily expressive, opening and closing to convey excitement, curiosity, or greeting. Fixed-position mounting with hot glue or screws through the chassis side plate works perfectly.

Route the servo cable from each gripper back toward the Pi's GPIO header. Label the cables with small pieces of tape — left gripper, right gripper — before routing, or you'll forget which is which.

### Step 8 — Wire Everything

This is where you spend the most careful time. Work through each subsystem one at a time and test before moving on.

**Motor driver to Pi GPIO:**

The DRV8833 has four input pins (AIN1, AIN2, BIN1, BIN2) and two motor output pairs (AOUT1/AOUT2 for the left track, BOUT1/BOUT2 for the right). Connect the inputs to Pi GPIO hardware PWM pins. GPIO 12 and 13 are hardware PWM on the Pi Zero and work reliably for motor control. Connect:

- AIN1 → GPIO 12
- AIN2 → GPIO 13
- BIN1 → GPIO 18
- BIN2 → GPIO 19
- VM (motor voltage) → 5V rail
- VCC (logic voltage) → 3.3V
- GND → ground

Connect the left track motor wires to AOUT1 and AOUT2, and the right track motor to BOUT1 and BOUT2.

**Eye servos:**

SG90 servos have three wires: brown (ground), red (5V), and orange or yellow (signal/PWM). Connect all four servo grounds to Pi ground and all four red wires to the 5V rail. Connect each signal wire to a separate GPIO pin:

- Left eye pan → GPIO 17
- Left eye tilt → GPIO 27
- Right eye pan → GPIO 22
- Right eye tilt → GPIO 23

**Gripper servos:**

Same wiring pattern as eye servos:

- Left gripper → GPIO 24
- Right gripper → GPIO 25

**Camera:**

Plug the CSI ribbon cable into the Pi Zero's camera connector. The connector has a small plastic latch — lift it gently, insert the cable with the contacts facing the board, then press the latch back down firmly. The cable should not pull out with gentle tugging.

**Microphone and speaker:**

If using a USB microphone, it plugs directly into the Pi Zero's micro USB OTG port via a micro-USB-to-USB-A adapter. If using an I2S MEMS microphone module, connect it to the I2S pins (GPIO 18 for BCLK, GPIO 19 for LRCLK, GPIO 20 for DATA — note these overlap with motor driver pins so verify your exact setup before finalising).

For the MAX98357 I2S amplifier and speaker: connect BCLK to GPIO 18, LRCLK to GPIO 19, DIN to GPIO 21, VIN to 5V, GND to ground. Connect your 3W speaker to the amplifier's + and − outputs.

**Power:**

Your 5V boost converter takes the 3.7V LiPo input and outputs a stable 5V at 2–3A. Connect the boost output to the Pi's 5V GPIO pin (pin 2 or 4) and to the motor/servo power rail. Confirm the boost converter is outputting exactly 5V before connecting the Pi — use a multimeter. Under-voltage will cause random crashes and over-voltage will destroy the Pi immediately.

**Test the wiring before powering on:** With the battery disconnected, use a multimeter in continuity mode to check that no GPIO pin is shorted to ground or to another GPIO pin through mis-routed wires.

---

## Part Four: Software Setup

### Step 9 — Flash the SD Card

Download the Raspberry Pi Imager from raspberrypi.com and install it on your main computer. Insert your micro SD card.

In the Imager, select "Raspberry Pi Zero 2 W" as the device, choose "Raspberry Pi OS Lite (64-bit)" as the operating system, and select your SD card as the storage target.

Before you click Write, click the gear icon (or press Ctrl+Shift+X) to open the Advanced Options panel. Here you must:

1. Set a hostname (e.g. `robot`)
2. Enable SSH and set a username and password
3. Enter your Wi-Fi SSID and password — **critical: your home network must be 2.4 GHz**. The Pi Zero 2 W does not have a 5 GHz radio. If your router broadcasts a combined 2.4/5 GHz network under one name, log into your router settings and split them into separate SSIDs.

Click Save, then Write. The process takes about five minutes. When it finishes, eject the card and insert it into the Pi.

### Step 10 — First Boot and SSH Connection

Connect the battery and power on the Pi. Wait about 45 seconds for it to boot. From your main computer on the same Wi-Fi network, open a terminal and run:

```
ssh pi@robot.local
```

If the hostname doesn't resolve, find the Pi's IP address from your router's DHCP client list and SSH to that address directly. Once you're in, run the following to update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

Then enable the camera interface:

```bash
sudo raspi-config
```

Navigate to Interface Options → Camera and enable it. Also enable I2C and SPI while you're here if you plan to use I2S audio. Reboot when prompted.

### Step 11 — Install Python Libraries

SSH back in after the reboot and install the required libraries:

```bash
sudo apt install -y python3-picamera2 python3-gpiozero python3-pip python3-dev flac
pip3 install SpeechRecognition gTTS playsound openai
```

If you plan to use Anthropic Claude as your LLM:

```bash
pip3 install anthropic
```

If you want to run a local LLM via Ollama, install it with:

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull phi3:mini
```

Note that Ollama will take several minutes to download the model and will use about 2 GB of SD card space. Response times on the Pi Zero 2 W will be slow (10–30 seconds per response) but it works without any internet connection.

### Step 12 — Test the Camera

Before writing any robot code, verify the camera works:

```bash
libcamera-still -o test.jpg
```

This should capture a photo and save it as `test.jpg`. Transfer it to your main computer with `scp pi@robot.local:test.jpg .` and open it. If the image is black, the CSI cable is not properly seated — power off, reseat the cable, and try again.

### Step 13 — Test the Servos

Create a test script to verify each servo responds correctly before integrating everything. Save the following as `servo_test.py`:

```python
from gpiozero import Servo
from time import sleep

# Test each servo one at a time
servo = Servo(17)  # change pin to test each servo

servo.min()
sleep(1)
servo.mid()
sleep(1)
servo.max()
sleep(1)
servo.mid()
```

Run it with `python3 servo_test.py`. The servo should sweep from its minimum to maximum position and return to centre. If it grinds or twitches violently at the extremes, the servo's physical range is being exceeded — you'll fix this in calibration (Part Five).

Repeat for each servo by changing the pin number. Write down the actual usable min/max pulse widths for each servo as you go — you'll need them.

### Step 14 — Test the Motors

Save the following as `motor_test.py`:

```python
from gpiozero import Motor
from time import sleep

left = Motor(forward=12, backward=13)
right = Motor(forward=18, backward=19)

# Drive forward
left.forward(0.5)
right.forward(0.5)
sleep(1)

# Stop
left.stop()
right.stop()
sleep(0.5)

# Spin in place
left.forward(0.5)
right.backward(0.5)
sleep(0.5)

left.stop()
right.stop()
```

Run it with `python3 motor_test.py`. The robot should drive forward briefly, stop, then spin. If it pulls to one side during the forward run, adjust the speed value for the stronger motor down slightly (e.g. change 0.5 to 0.45) until it tracks straight.

If a motor spins the wrong direction, swap its two output wires at the DRV8833 terminal — no code changes needed.

### Step 15 — Test the Microphone and Speaker

Test audio recording:

```bash
arecord -D plughw:1,0 -f cd -t wav -d 3 test.wav
```

This records three seconds of audio. Play it back:

```bash
aplay test.wav
```

You should hear your own voice. If `arecord` complains about the device, run `arecord -l` to list available recording devices and use the correct card and device numbers.

Test the speaker and text-to-speech with a short Python snippet:

```python
from gtts import gTTS
import os

tts = gTTS("Hello, I am your robot. I am awake and ready.")
tts.save("hello.mp3")
os.system("mpg123 hello.mp3")
```

Install mpg123 if needed: `sudo apt install mpg123`.

---

## Part Five: The Robot Brain — Writing the Core Loop

### Step 16 — Write the Main Script

This is the heart of the project. Create a file called `robot.py` on the Pi. The script does five things in a continuous loop: it listens for your voice, converts it to text, sends it to the LLM, parses the JSON response, and executes the physical actions.

```python
import json
import os
import time
import base64
import threading
import speech_recognition as sr
from gtts import gTTS
from gpiozero import Servo, Motor
from picamera2 import Picamera2
import openai  # or import anthropic

# --- Configuration ---
OPENAI_API_KEY = "your-api-key-here"
openai.api_key = OPENAI_API_KEY

SYSTEM_PROMPT = """
You are a small tracked robot. You have two eyes that can pan and tilt,
two small claw hands that open and close, a camera to see with, and tracks
to move on. You are curious, friendly, and expressive. You animate your
eyes and hands to match your emotions.

You MUST always respond in this exact JSON format and nothing else:
{
  "spoken": "what you want to say out loud",
  "eyes": {"left_pan": 0, "left_tilt": 0, "right_pan": 0, "right_tilt": 0},
  "hands": {"left": "open", "right": "open"},
  "move": "none"
}

Eye values are degrees from centre: range -45 to +45.
Hand values are either "open" or "close".
Move values are: "none", "forward", "backward", "left", "right".
"""

# --- Hardware Setup ---
left_motor  = Motor(forward=12, backward=13)
right_motor = Motor(forward=18, backward=19)

# Eye servos — adjust min_pulse/max_pulse to your calibrated values
eye_left_pan   = Servo(17, min_pulse_width=0.0008, max_pulse_width=0.0022)
eye_left_tilt  = Servo(27, min_pulse_width=0.0008, max_pulse_width=0.0022)
eye_right_pan  = Servo(22, min_pulse_width=0.0008, max_pulse_width=0.0022)
eye_right_tilt = Servo(23, min_pulse_width=0.0008, max_pulse_width=0.0022)

# Gripper servos
hand_left  = Servo(24, min_pulse_width=0.0008, max_pulse_width=0.0022)
hand_right = Servo(25, min_pulse_width=0.0008, max_pulse_width=0.0022)

camera = Picamera2()
camera.configure(camera.create_still_configuration())
camera.start()
time.sleep(1)  # let camera warm up

recogniser = sr.Recognizer()
mic = sr.Microphone()

# --- Helper Functions ---

def speak(text):
    tts = gTTS(text)
    tts.save("/tmp/response.mp3")
    os.system("mpg123 -q /tmp/response.mp3")

def take_photo_base64():
    camera.capture_file("/tmp/snapshot.jpg")
    with open("/tmp/snapshot.jpg", "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")

def degrees_to_servo_value(degrees):
    """Convert -45 to +45 degrees into gpiozero servo value -1 to +1."""
    return max(-1.0, min(1.0, degrees / 45.0))

def move_robot(direction, duration=0.4, speed=0.5):
    if direction == "forward":
        left_motor.forward(speed);  right_motor.forward(speed)
    elif direction == "backward":
        left_motor.backward(speed); right_motor.backward(speed)
    elif direction == "left":
        left_motor.backward(speed); right_motor.forward(speed)
    elif direction == "right":
        left_motor.forward(speed);  right_motor.backward(speed)
    else:
        return
    time.sleep(duration)
    left_motor.stop()
    right_motor.stop()

def animate(action):
    eyes = action.get("eyes", {})
    hands = action.get("hands", {})
    direction = action.get("move", "none")

    eye_left_pan.value   = degrees_to_servo_value(eyes.get("left_pan", 0))
    eye_left_tilt.value  = degrees_to_servo_value(eyes.get("left_tilt", 0))
    eye_right_pan.value  = degrees_to_servo_value(eyes.get("right_pan", 0))
    eye_right_tilt.value = degrees_to_servo_value(eyes.get("right_tilt", 0))

    hand_left.value  = 1.0 if hands.get("left", "open") == "open" else -1.0
    hand_right.value = 1.0 if hands.get("right", "open") == "open" else -1.0

    if direction != "none":
        move_robot(direction)

def ask_llm(user_text, include_photo=False):
    messages = [{"role": "system", "content": SYSTEM_PROMPT}]

    if include_photo:
        image_data = take_photo_base64()
        messages.append({
            "role": "user",
            "content": [
                {"type": "text", "text": user_text},
                {"type": "image_url", "image_url": {
                    "url": f"data:image/jpeg;base64,{image_data}"
                }}
            ]
        })
    else:
        messages.append({"role": "user", "content": user_text})

    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        max_tokens=300
    )
    return response.choices[0].message.content

# --- Main Loop ---

def main():
    speak("Hello! I am awake and ready.")
    print("Robot ready. Press Ctrl+C to stop.")

    while True:
        with mic as source:
            recogniser.adjust_for_ambient_noise(source, duration=0.5)
            print("Listening...")
            try:
                audio = recogniser.listen(source, timeout=5, phrase_time_limit=8)
            except sr.WaitTimeoutError:
                continue

        try:
            user_text = recogniser.recognize_google(audio)
            print(f"You said: {user_text}")
        except sr.UnknownValueError:
            print("Could not understand audio")
            continue
        except sr.RequestError as e:
            print(f"STT error: {e}")
            continue

        # Check if the user wants vision
        wants_photo = any(word in user_text.lower() for word in
                          ["see", "look", "what is", "what's", "show"])

        try:
            raw = ask_llm(user_text, include_photo=wants_photo)
            action = json.loads(raw)
        except (json.JSONDecodeError, Exception) as e:
            print(f"LLM/parse error: {e}")
            speak("Sorry, I got confused. Can you say that again?")
            continue

        spoken = action.get("spoken", "")
        print(f"Robot says: {spoken}")

        # Animate and speak concurrently
        anim_thread = threading.Thread(target=animate, args=(action,))
        anim_thread.start()
        speak(spoken)
        anim_thread.join()

if __name__ == "__main__":
    main()
```

Save this file, then run it:

```bash
python3 robot.py
```

Speak to the robot. It should respond with spoken audio and physical movement.

---

## Part Six: Calibration

### Step 17 — Calibrate the Eye Servos

Out of the box, the `Servo` class in gpiozero uses pulse widths of 1ms (min) to 2ms (max). Many SG90 servos actually respond to a wider range of 0.5ms to 2.5ms, but your eye mechanism may only have physical travel over part of that range. If you commanded the servo to its absolute limit and heard it grinding in Step 13, you need to narrow the software range.

Edit the `min_pulse_width` and `max_pulse_width` values in the Servo constructor in `robot.py`. Start with `0.0009` and `0.0021` and test. Increase or decrease in steps of `0.0001` (0.1ms) until the eye moves through its full intended range without hitting a mechanical stop.

Centre position (servo value 0.0) should correspond to the eye looking straight forward. Check this by running:

```python
from gpiozero import Servo
s = Servo(17, min_pulse_width=0.0009, max_pulse_width=0.0021)
s.mid()
```

If the eye is not centred at `mid()`, physically adjust the servo horn position while the servo is powered at centre — this is the one mechanical adjustment that cannot be done in software.

### Step 18 — Tune Motor Straight-Line Driving

Lay the robot on the floor and run the forward test from Step 14. Watch it drive across the room. If it curves, one motor is stronger than the other. Reduce the speed value for the faster motor in the `move_robot()` function until the robot drives straight. A 0.05 difference in speed value is usually enough to correct most motors. Write down your final calibrated speed values.

---

## Part Seven: Testing the Full System

### Step 19 — Full Integration Test

With `robot.py` running, work through these test interactions one by one and verify each:

**Basic response test:** Say "Hello, how are you?" The robot should speak a response, and its eyes should move to an appropriate position (looking slightly up and to one side when thinking, for example — the LLM will choose this naturally).

**Emotion test:** Say "I have great news!" The robot should respond with an enthusiastic expression — eyes wide (tilted forward), hands open, possibly moving forward slightly.

**Vision test:** Hold an object in front of the camera and say "What can you see?" The robot should pause for a moment while it captures a photo and sends it to the LLM, then describe what it sees.

**Movement test:** Say "Come closer to me." The robot should say something and drive forward briefly.

**Gripper test:** Say "Wave hello to me." The robot should open and close one or both grippers while speaking.

### Step 20 — Handling Edge Cases

After basic testing works, you'll likely encounter a few rough edges. Here's how to handle the most common ones.

**The LLM occasionally returns plain text instead of JSON.** This happens when the model ignores your format instruction. Add a stricter instruction to the system prompt: `"You must respond ONLY with valid JSON. Do not include any other text, explanation, or markdown code blocks."` If it still fails occasionally, the `json.loads()` try/except in the main loop will catch it and ask the user to repeat themselves.

**The robot's eye movement looks jumpy.** The servos are snapping directly to their target position. Smooth this out by interpolating between current and target position in small steps. Replace the direct servo assignment in `animate()` with a short loop that moves the servo value by 0.1 steps every 20ms.

**Response latency is too long.** If you're using GPT-4o, switch to `gpt-4o-mini` — it's roughly four times faster and still produces good JSON output for this use case. If you're using Claude, `claude-haiku-4-5` is the fastest option. For local Ollama, `phi3:mini` is the best balance of speed and quality on the Pi Zero 2 W.

**The microphone picks up the robot's own voice and triggers a new loop.** Add a short `time.sleep(0.5)` after the `speak()` call returns, before the next `recogniser.listen()` call. This gives the audio hardware time to stop outputting before the mic starts listening again.

---

## Taking It Further

Once the robot is working reliably, there are several natural next steps.

**Add a wake word.** Instead of always-on listening, use the `pvporcupine` library from Picovoice to listen for a custom wake word like "Hey Robot" before activating the main loop. This dramatically reduces CPU load and accidental activations.

**Give the robot a persistent memory.** Save the LLM conversation history to a JSON file and reload it on startup. The robot will remember things you told it in previous sessions, which makes interactions feel much more personal.

**Add obstacle avoidance.** A single HC-SR04 ultrasonic sensor costs about $2 and gives the robot the ability to stop before driving off a table. Wire the trigger pin to GPIO 5 and the echo pin to GPIO 6, and add a distance check before any forward movement command is executed.

**Improve the voice.** `gTTS` gets the job done but sounds robotic. The ElevenLabs API offers extremely natural-sounding voices with a generous free tier. Swap out the `speak()` function to call the ElevenLabs API instead and the difference is immediately noticeable.

---

## Troubleshooting Quick Reference

| Symptom | Most Likely Cause | Fix |
|--------|------------------|-----|
| Pi won't connect to Wi-Fi | 5 GHz network | Split router SSIDs; connect to 2.4 GHz only |
| Camera returns black image | CSI cable not seated | Power off, reseat cable, check ZIF latch |
| Servo grinds at limits | Pulse width too wide | Reduce max_pulse_width in 0.1ms steps |
| Robot pulls left or right | Motor speed mismatch | Reduce faster motor speed by 0.05 |
| LLM returns plain text not JSON | Model ignoring format | Strengthen JSON instruction in system prompt |
| Servo jitter at rest | Voltage ripple | Add 10µF cap on servo power rail |
| Pi crashes under load | Under-voltage | Check boost converter output; must be stable 5V at 2A |
| Speech recognition fails | Wrong mic device | Run `arecord -l` to find correct device index |
| No sound from speaker | I2S not enabled | Run `raspi-config`, enable I2S; reboot |
| Response latency too high | Slow LLM model | Switch to a faster model endpoint or reduce max_tokens |
