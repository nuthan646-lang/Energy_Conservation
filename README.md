# Green Building Digital Twin

> Computer vision tracks people in a building, simulated lights switch on ahead of them and dim behind them, and generative AI turns the data into energy and sustainability recommendations.

Built for the **Green Building Digital Twin** problem statement: *develop a digital twin of a building that integrates computer vision to monitor occupancy, lighting and resource usage, with generative AI that analyzes the data and recommends improvements to energy efficiency, indoor sustainability and resource utilization.*

**Tags:** `computer-vision` `digital-twin` `genai` `smart-buildings` `energy-efficiency` `opencv` `flask`

---

## The idea

Game engines only render what the player is about to see. This project applies the same idea to lighting: instead of lighting a whole corridor, only the lights **ahead** of a walking person are fully on. The lights behind them fade out, and a zone switches off a few seconds after the last person leaves.

There is no lighting hardware in this version. Each zone has 30 **simulated lights**, shown on a live dashboard, and all energy figures are estimates.

## Features

| Requirement | What the project does |
|---|---|
| **Occupancy monitoring** | Detects and tracks up to 4 moving people per zone with position, speed and direction. Shows people count per zone, for the whole building, and as a zone breakdown. |
| **Lighting** | Lights ahead of a walker turn on, with a lead distance that grows with speed. Lights behind fade out, a low ambient level stays on briefly after the last detection, then the zone switches off. |
| **Resource usage** | Compares live power with an always-on baseline and shows percent saved, Wh saved, CO₂ avoided, an extrapolated monthly cost saving and a power trend chart (2, 5 or 15 minutes). |
| **Digital twin** | A 3D isometric floor plan with live lighting intensity per zone, people markers with movement direction, camera positions and zone boundaries, next to the live camera feeds. |
| **Generative AI** | Recommendations in three categories: energy efficiency, indoor sustainability and resource utilization. Uses Gemini or a local Ollama model, with a rule-based fallback when no AI is configured. |
| **Security alerts** | Flags occupancy detected during configurable after-hours as an *unverified* alert, and flags offline cameras. |
| **Multi-camera, multi-user** | Up to 3 camera zones. Friends can contribute a webcam from any network, and anyone with the link can watch the dashboard. |
| **Reports** | One-click text snapshot report of the current building state. |

## How it works

```
 webcam / phone / friend's browser
              |
              v
   OpenCV motion detection  ->  per-person tracking (position, speed, direction)
              |
              v
   simulated light rendering (ahead = full, behind = fading, vacancy rules)
              |
              v
   energy estimates  ->  Flask API (/state)  ->  dashboard + GenAI recommendations
```

One background thread per camera does the vision work, so many viewers can open the dashboard without extra load on the cameras. If a camera goes offline, its zone is held at a safe low light level rather than treated as empty.

## Quick start

```bash
git clone <your-repo-url>
cd <your-repo>
pip install flask opencv-python numpy requests
python tracker.py
```

Open `http://localhost:5000`. Camera 0 is your laptop webcam.

To demo with no cameras at all, run in demo mode. A simulated person then walks through every zone:

```bash
# Windows PowerShell
$env:DEMO_MODE="1"; python tracker.py
# macOS / Linux
DEMO_MODE=1 python tracker.py
```

## Adding more cameras

Edit `CAMERAS` in `tracker.py`. Each zone can use:

- `0` for a local webcam
- `"http://phone-ip:8080/video"` for a phone running an IP-webcam app
- `"remote"` for a camera sent over the network
- `"sim"` for the built-in simulation

### Remote cameras from any network

1. Keep `python tracker.py` running.
2. Start a tunnel, for example: `cloudflared tunnel --url http://localhost:5000 --protocol http2`
3. Send each contributor your tunnel link plus `/cam/1` or `/cam/2`.
4. They open it, press **Start sending camera** and allow webcam access. Browsers only allow webcam access on HTTPS pages, which the tunnel provides.

On the same Wi-Fi you can instead run `HTTPS=1 python tracker.py` (needs `pip install cryptography`), or run `send_cam.py` on the other laptop:

```bash
python send_cam.py http://HOST_IP:5000 1
```

## Generative AI setup (optional)

```bash
# Gemini
set AI_BACKEND=gemini
set GEMINI_API_KEY=your-key
# or a local model
set AI_BACKEND=ollama
```

Use `export` instead of `set` on macOS or Linux. Without these, the dashboard uses built-in rules to generate the recommendations. Check `GEMINI_MODEL` in `tracker.py` against the models currently available to you.

## Configuration

Constants at the top of `tracker.py`:

| Setting | Meaning |
|---|---|
| `AMBIENT`, `VACANCY_S` | Low light level kept after the last detection, and how many seconds before the zone switches off |
| `MIN_AREA`, `MOVE_V` | How large a moving shape must be to count as a person, and how fast it must move to count as walking |
| `WATT_PER_LIGHT`, `CO2_KG_PER_KWH`, `RS_PER_KWH` | Assumptions behind the energy, CO₂ and cost estimates |

Environment variables:

| Variable | Default | Meaning |
|---|---|---|
| `DEMO_MODE` | `0` | `1` uses simulated cameras everywhere |
| `FADE_OUT_S` | `1.5` | How slowly lights fade out |
| `OFFLINE_SAFE_LEVEL` | `0.25` | Light level held when a camera is offline |
| `SECURITY_ENABLED`, `SECURITY_START_HOUR`, `SECURITY_END_HOUR` | `1`, `22`, `6` | After-hours alert window |
| `AI_BACKEND` | `rules` | `rules`, `gemini` or `ollama` |
| `HTTPS` | unset | `1` serves over HTTPS with a self-signed certificate |

## API

| Endpoint | Purpose |
|---|---|
| `GET /state` | Live data for every zone, building totals and the latest AI insight |
| `GET /video_feed/<id>` | MJPEG stream with tracking overlay |
| `POST /api/remote_cam/<id>` | Receive JPEG frames from a remote camera |
| `GET /cam/<id>` | Browser page that sends a webcam to zone `<id>` |
| `GET /api/security/events` | Logged after-hours occupancy alerts |
| `POST /api/ai/run` | Run the AI analysis now |

## Assumptions and limitations

- **Lights are simulated.** The numbers are estimates based on assumed values: 10 W per light, an always-on baseline, 0.82 kg CO₂/kWh and ₹8/kWh. The lighting output is a dry run, and nothing controls real lights.
- **Detection is motion-based**, not a trained person-recognition model. People who stand very still for a long time may fade into the background, and people close together may be counted as one.
- **Security alerts are unverified.** They report that movement was seen during after-hours, not that an intruder was confirmed.
- **Indoor comfort** is shown as unavailable because there is no temperature sensor.
- **No authentication.** Anyone with the link can open the dashboard and send frames, so share it only with people you trust.
- Check everything with your own cameras and devices before relying on it.

## Roadmap

- Drive real lights (for example WLED or ESP32 strips) from the same light values, after testing the controller separately.
- Add an ambient light sensor so lights dim or switch off when daylight is sufficient.
- Add a temperature and air-quality sensor for the indoor comfort reading.
- Add other resources to the twin, such as HVAC and water.
- Replace motion detection with a person-detection model for more reliable counts.

## Project structure

```
tracker.py     Flask server, computer vision, light and energy simulation, GenAI
index.html     Dashboard (3D digital twin, cameras, trends, alerts, recommendations)
send_cam.py    Optional script to send a laptop webcam to the server
```
## License
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.