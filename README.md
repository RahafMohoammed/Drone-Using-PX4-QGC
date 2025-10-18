# Drone-Using-PX4-QGC
Guidance Drone — AlUla Tour Guide (PX4 + MAVSDK)  
✨ Features

PX4 SITL + MAVSDK (Python) control

Waypoint navigation with arrival checks

Optional voice narration (TTS)

Works with QGroundControl for monitoring

🗺️ Landmarks (default)

Elephant Rock: 26.689204, 37.981599

Jabal Al-Jurra: 26.6818181, 37.9794269

Custom launch (blue pin): 26.6857562, 37.9802661

🧱 Repo Structure (suggested)
guidance-drone/
├─ README.md
├─ LICENSE
├─ requirements.txt
├─ scripts/
│  ├─ simple_tour.py                 # super simple tour (goto + waits)
│  ├─ simple_tour_with_tts.py        # same + voice narration
│  └─ mission_tour.py                # Mission API version (loiter + RTL)
└─ utils/
   └─ tts.py                         # gTTS/pyttsx3 helpers (optional)

📦 Requirements

PX4 SITL (jmavsim or Gazebo)

Python 3.9+

MAVSDK-Python

(Optional) gTTS + playsound or pyttsx3 for offline TTS

QGroundControl (monitoring)

requirements.txt example:

mavsdk>=2.5.0
pyttsx3>=2.90    # optional (offline)
gTTS>=2.5.1      # optional (online)
playsound==1.2.2 # optional

🚀 Quick Start
1) Start PX4 at the blue-pin Home (AlUla)
# in PX4-Autopilot directory
PX4_HOME_LAT=26.6857562 PX4_HOME_LON=37.9802661 PX4_HOME_ALT=100 make px4_sitl jmavsim

2) Open QGroundControl

Just for map visualization and telemetry.

3) Create & activate a venv
python -m venv .venv
source .venv/bin/activate         # Windows: .venv\Scripts\activate
pip install -r requirements.txt

4) Run the tour script

Use udpin:// (recommended; udp:// is deprecated)

python scripts/simple_tour.py

📜 Example: scripts/simple_tour.py

A very short tour: connect → arm → takeoff → go to Elephant → wait/speak → go to Jurra → wait/speak → RTL/Land.

import asyncio, time, math
from mavsdk import System

LAUNCH_LAT, LAUNCH_LON = 26.6857562, 37.9802661
ELEPHANT = (26.689204, 37.981599, "Elephant Rock")
JURRA    = (26.6818181, 37.9794269, "Jabal Al-Jurra")
CLIMB_ABOVE_HOME = 25.0
ARRIVAL_TOL_M = 20.0
TIMEOUT_S = 180.0
HOVER_S = 30
VISITOR_OFFSET_M = 50.0  # 50 m "in front" (north)

def m2lat(m): return m/111_320.0
def hav(lat1, lon1, lat2, lon2):
    import math
    R=6371000.0; dlat=math.radians(lat2-lat1); dlon=math.radians(lon2-lon1)
    a=math.sin(dlat/2)**2+math.cos(math.radians(lat1))*math.cos(math.radians(lat2))*math.sin(dlon/2)**2
    return 2*R*math.asin(math.sqrt(a))

async def goto_and_wait(drone, lat, lon, alt, tol=ARRIVAL_TOL_M, to=TIMEOUT_S):
    await drone.action.goto_location(lat, lon, alt, yaw_deg=float('nan'))
    t0=time.time(); last=None; still=0
    async for p in drone.telemetry.position():
        d=hav(p.latitude_deg,p.longitude_deg,lat,lon)
        print(f"  dist: {d:.1f} m")
        still = still+1 if (last is not None and abs(d-last)<0.7) else 0
        last=d
        if d<=tol or still>80: return True
        if time.time()-t0>to: return False
        await asyncio.sleep(0.3)

async def main():
    drone=System()
    print("Connecting udpin://:14540 ...")
    await drone.connect(system_address="udpin://:14540")

    async for s in drone.core.connection_state():
        if s.is_connected: break
    async for h in drone.telemetry.health():
        if h.is_global_position_ok and h.is_home_position_ok: break

    home=None
    async for hh in drone.telemetry.home():
        home=hh; break
    flight_alt=home.absolute_altitude_m+CLIMB_ABOVE_HOME

    try: await drone.action.set_maximum_speed(10.0)
    except: pass

    await drone.action.arm(); await asyncio.sleep(1)
    await drone.action.takeoff(); await asyncio.sleep(6)

    # Go to launch pin first
    await goto_and_wait(drone, LAUNCH_LAT, LAUNCH_LON, flight_alt)

    # Visitor near Elephant (50 m north)
    el_lat, el_lon, _ = ELEPHANT
    v1_lat, v1_lon     = el_lat + m2lat(VISITOR_OFFSET_M), el_lon
    print("Elephant spot..."); await goto_and_wait(drone, v1_lat, v1_lon, flight_alt)
    print("Narration: Elephant Rock ..."); await asyncio.sleep(HOVER_S)

    # Visitor near Jurra (50 m north)
    ju_lat, ju_lon, _ = JURRA
    v2_lat, v2_lon    = ju_lat + m2lat(VISITOR_OFFSET_M), ju_lon
    print("Jurra spot..."); await goto_and_wait(drone, v2_lat, v2_lon, flight_alt)
    print("Narration: Jabal Al-Jurra ..."); await asyncio.sleep(HOVER_S)

    await drone.action.return_to_launch()
    await asyncio.sleep(5)

if __name__ == "__main__":
    asyncio.run(main())


For narration (TTS), see scripts/simple_tour_with_tts.py (gTTS/pyttsx3).
On WSL, consider using Windows PowerShell TTS bridge.

🧩 Mission API Variant

If you prefer a robust mission (waypoints + loiter + RTL even if your script stops), use scripts/mission_tour.py with MissionItem/MissionPlan.

🪪 Notes (Windows / WSL)

Use udpin://:14540 in Python.

Ensure PX4 sends MAVLink to port 14540 (Onboard link).

On WSL, audio playback may not work—use pyttsx3 on Windows or a PowerShell TTS bridge.

🧰 Troubleshooting

Stuck waiting to arrive → increase ARRIVAL_TOL_M to 20–25 m; ensure speed is set (set_maximum_speed(10.0)).

QGC shows old brown path → Clear mission in QGC (Plan → Clear) or await drone.mission.clear_mission().

No voice → switch to pyttsx3 (offline), or run script on Windows (not WSL) with PowerShell TTS.

🗺️ Roadmap Ideas

Multilingual narration (Arabic/English toggle)

Photo/video capture and QR share

Orbit mode around POIs during narration

Mission editor UI for custom routes

🙏 Acknowledgments

Tuwaiq Academy

Mr. Abdulkarim for mentorship and support

📄 License

MIT — see LICENSE.
