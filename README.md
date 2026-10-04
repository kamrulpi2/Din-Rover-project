# Smart Monitor: ESP32 + Firebase IoT Dashboard

An ESP32 reads gas, air, soil, water, IR and distance sensors and sends the data to **Firebase Realtime Database**. A single-file web dashboard (`index.html`) shows the data live, controls two servos from buttons, and keeps a team and parts list.

## Features

- **Live sensor dashboard**
  - 3 IR sensors
  - Soil moisture and water level
  - 6 gas sensors: MQ-2, MQ-5, MQ-7, MQ-8, MQ-9, MQ-135
  - Temperature, humidity and pressure (BME280)
  - Dust and ultrasonic distance
- **Mini graphs, min/max and level badges** on every sensor card
- **Servo control buttons** that write to Firebase, which the ESP32 reads
- **Rover drive pad** with speed, auto mode and light (writes to `/control`)
- **Live data tab** that can watch any Firebase path, with a 50-row data log and CSV download
- **Alerts** for high temperature, a close obstacle, low battery and hazards
- **Local time, live location, weather and a Bangladesh map**
- **Team page** where you can add, edit and delete members and their project parts
- One HTML file, no build step

## Hardware

| Part | Quantity |
|------|----------|
| ESP32 dev board | 1 |
| IR obstacle sensor | 3 |
| Soil moisture sensor (analog) | 1 |
| Water level sensor (analog) | 1 |
| MQ-2, MQ-5, MQ-7, MQ-8, MQ-9, MQ-135 gas sensors | 1 each |
| Dust sensor (Sharp GP2Y10 type) | 1 |
| BME280 (I2C) | 1 |
| Ultrasonic sensor (HC-SR04) | 1 |
| Servo (S1: 0 to 180 deg) and 360 deg servo (S2) | 1 each |

## Pin map

| Sensor | GPIO | Sensor | GPIO |
|--------|------|--------|------|
| IR 1 | 2 | MQ-7 | 33 |
| IR 2 | 3 | MQ-8 | 32 |
| IR 3 | 13 | MQ-9 | 27 |
| Soil moisture | 32 | MQ-135 | 35 |
| Water level | 34 | Dust LED | 4 |
| Servo 1 | 16 | Dust VO | 25 |
| Servo 2 | 17 | Ultrasonic TRIG | 5 |
| MQ-2 | 34 | Ultrasonic ECHO | 18 |
| MQ-5 | 26 | BME280 SDA / SCL | 21 / 22 |

### Known pin issues (read before wiring)

1. **Two pins are used twice.** Soil moisture and MQ-8 both use GPIO 32. Water level and MQ-2 both use GPIO 34. Move soil moisture to **GPIO 36** and water level to **GPIO 39**. Both are free ADC1 pins.
2. **ADC2 pins do not read while WiFi is on.** MQ-5 (26), MQ-9 (27) and Dust VO (25) are ADC2 pins, so `analogRead()` returns wrong values (often 0). The ESP32 has only six ADC1 pins (32, 33, 34, 35, 36, 39), and they are already used by MQ-2, MQ-7, MQ-8, MQ-135, soil and water. To read MQ-5, MQ-9 and dust correctly, add an external I2C ADC such as the **ADS1115**.
3. **GPIO 2 and GPIO 3.** GPIO 3 is the Serial RX pin, and GPIO 2 is a boot pin. Disconnect IR 1 and IR 2 while uploading code if the upload fails.
4. **Ultrasonic ECHO is 5 V.** Use a voltage divider (for example 1 k and 2 k ohms) before GPIO 18, because the ESP32 is 3.3 V only.
5. **Gas sensors need warm-up.** MQ sensors need several minutes to heat up, and they need a 5 V supply.

## Servo behavior

| Firebase value | Servo 1 | Servo 2 |
|----------------|---------|---------|
| `Control/Servo1` = `1` | moves 0 to 180 deg | |
| `Control/Servo1` = `0` | moves 180 to 0 deg | |
| `Control/Servo2` = `1` | | moves 0 to 360 deg |
| `Control/Servo2` = `0` | | moves 360 to 0 deg |

Servo 2 assumes a 360 deg positional servo. A normal servo only turns 180 deg.

## Firebase data structure

```json
{
  "SensorData": {
    "IR1": 0, "IR2": 1, "IR3": 0,
    "SoilMoisture": 40, "WaterLevel": 70,
    "MQ2": 17, "MQ5": 60, "MQ7": 19, "MQ8": 69, "MQ9": 79, "MQ135": 5,
    "Dust": 0, "Temperature": 28.5, "Humidity": 62, "Pressure": 1008,
    "Distance": 35
  },
  "Control": { "Servo1": 0, "Servo2": 0 },
  "control": { "cmd": "stop", "speed": 150, "auto": false, "light": false },
  "Team": {
    "-Nabc123": { "name": "", "role": "", "part": "", "phone": "", "email": "", "photo": "" }
  }
}
```

- Gas sensors are mapped to a **0 to 100** scale.
- `Distance` is `-1` when there is no echo.
- A BME280 that is not found uploads `0` for temperature, humidity and pressure. The dashboard shows "Sensor not found".
- Paths are case-sensitive. `Control` (servos) and `control` (rover drive) are two separate paths.

## Getting started

### 1. Firebase

1. Create a project in the [Firebase console](https://console.firebase.google.com/).
2. Add a **Realtime Database** and choose your region.
3. Go to **Authentication > Sign-in method** and enable **Email/Password**. Create the user the ESP32 will sign in with.
4. Set the database rules. This is for testing only (see the security note below):

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

### 2. ESP32

1. Install the Arduino IDE with ESP32 board support.
2. Install these libraries from the Library Manager:
   - `FirebaseClient` by Mobizt
   - `ESP32Servo` by Kevin Harrington
   - `Adafruit BME280 Library` and `Adafruit Unified Sensor`
3. Open the sketch and fill in your own values:

```cpp
#define WIFI_SSID     "your-wifi-name"
#define WIFI_PASSWORD "your-wifi-password"
#define Web_API_KEY   "your-firebase-web-api-key"
#define DATABASE_URL  "https://your-project-default-rtdb.region.firebasedatabase.app/"
#define USER_EMAIL    "your-firebase-user@example.com"
#define USER_PASS     "your-firebase-user-password"
```

4. Fix the pin issues above, select your board and upload.

### 3. Dashboard

1. Open `index.html` in a text editor and replace `firebaseConfig` with your own project's config (Firebase console > Project settings > Your apps).
2. Open `index.html` in a browser. It needs an internet connection for Firebase, icons, fonts, the map and the weather.

### 4. Host it (optional)

- **GitHub Pages:** Settings > Pages > deploy from the main branch.
- **Firebase Hosting:** run `firebase init hosting` and then `firebase deploy`.

Firebase Analytics only runs on hosted `http` or `https` pages, so it is skipped when you open the file directly.

## Project structure

```
.
├── index.html     # Web dashboard (HTML + CSS + JS, Firebase SDK from CDN)
├── smart_monitor/ # ESP32 Arduino sketch (.ino)
└── README.md
```

## Security note

- **Do not commit real WiFi passwords or Firebase user credentials.** Use placeholders in the repository.
- Open rules (`.read` and `.write` set to `true`) let anyone with your database URL read and change your data. Use them only while testing. For real use, require sign-in, for example `".read": "auth != null"`, and sign in from the web page.
- The Firebase web API key is not secret, but your rules decide who can access the data.

## Tech stack

ESP32, Arduino C++, Firebase Realtime Database, HTML, CSS, JavaScript, Leaflet, Font Awesome, Open-Meteo.

## License

Add the license you prefer, for example MIT, as a `LICENSE` file.
<img width="1566" height="820" alt="image" src="https://github.com/user-attachments/assets/65c853b3-85f3-4aff-ac79-1484d9734e47" />

<img width="1591" height="907" alt="image" src="https://github.com/user-attachments/assets/6f860ced-d27f-47d1-8942-35fae610b312" />

<img width="1846" height="897" alt="image" src="https://github.com/user-attachments/assets/e0557ae4-3142-4e75-837b-b8cfff596666" />

