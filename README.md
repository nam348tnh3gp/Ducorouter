# ESP32 Duino-Coin Miner - GOD TIER v17

A high-performance, production-grade ESP32 cryptocurrency miner for [Duino-Coin](https://duinocoin.com) with advanced features including NAT routing, thermal management, dual-core mining, and a web-based dashboard.

## Features

### Mining Performance
- **Dual-Core Mining**: Utilizes both ESP32 cores for maximum hashrate
- **Optimized SHA1 Implementation**: Custom `DSHA1` class with GCC `-Ofast` optimizations
- **Adaptive Difficulty**: Automatically adjusts mining difficulty based on accept rate
- **Dynamic Batch Submission**: Adjusts batch size based on network ping for optimal throughput
- **Lock-Free Ring Buffer**: Uses canonical seqlock implementation for thread-safe share buffering

### Network & Connectivity
- **NAT Router Mode**: Acts as a WiFi router, allowing other devices to share internet through the ESP32
- **Smart Node Selection**: Automatically selects the best Duino-Coin pool node
- **DNS Caching**: Reduces latency with 10-minute DNS cache TTL
- **Auto Node Switching**: Switches nodes based on ping quality and reject rate
- **mDNS Support**: Access device via `esp32miner.local`

### Thermal Management
- **PID Temperature Controller**: Maintains optimal operating temperature (target: 65°C, max: 80°C)
- **Predictive Throttling**: Uses temperature trend analysis to prevent overheating
- **Mining Intensity Control**: Dynamically adjusts workload based on temperature and WiFi quality

### Reliability & Monitoring
- **Task Supervisor**: Monitors and auto-restarts stuck tasks
- **Watchdog Timer**: 20-second hardware watchdog prevents system hangs
- **Persistent Statistics**: Saves total shares/accepted counts across reboots
- **WiFi Auto-Reconnect**: Handles connection drops with automatic recovery
- **Health Heartbeats**: Tracks task health for all mining and system tasks

### Web Dashboard
- **Real-Time Monitoring**: View hashrate, shares, temperature, and system status
- **WiFi Configuration**: Scan and connect to networks via web interface
- **AP Configuration**: Customize access point name and password
- **JSON API**: Machine-readable mining statistics at `/mining` endpoint
- **Captive Portal**: Auto-redirects new connections to configuration page

### Additional Features
- **OTA Updates**: Update firmware over WiFi using ArduinoOTA
- **LED Status Indication**: Visual feedback for mining activity
- **Display Support**: Optional SSD1306 OLED or 16x2 LCD display
- **IoT Sensors**: Support for DS18B20, DHT11/22, HSU07M temperature sensors

## Hardware Requirements

- ESP32 development board (dual-core recommended)
- USB cable for initial programming
- Optional: SSD1306 OLED display or 16x2 LCD
- Optional: Temperature sensor (DS18B20, DHT11/22)

## Installation

1. Install [Arduino IDE](https://www.arduino.cc/en/software) or PlatformIO
2. Install ESP32 board support
3. Install required libraries:
   - `ArduinoJson`
   - `ArduinoOTA` (built-in)
   - `WiFi` (built-in for ESP32)
   - Optional: `U8g2lib` for OLED display
   - Optional: `OneWire` and `DallasTemperature` for DS18B20
   - Optional: `DHT sensor library` for DHT sensors

4. Configure `Settings.h`:
   ```cpp
   extern char *DUCO_USER = "your_username";
   extern char *MINER_KEY = "your_mining_key";  // or "None"
   extern char *RIG_IDENTIFIER = "your_rig_name";
   extern const char SSID[] = "your_wifi_name";
   extern const char PASSWORD[] = "your_wifi_password";
   ```

5. Upload to ESP32

## Configuration

### Settings.h Options

| Option | Description |
|--------|-------------|
| `DUCO_USER` | Your Duino-Coin username |
| `MINER_KEY` | Mining key (set in wallet) or "None" |
| `RIG_IDENTIFIER` | Custom rig name or "Auto" |
| `LED_BLINKING` | Enable/disable LED activity indicator |
| `SERIAL_PRINTING` | Enable/disable serial debug output |
| `DISPLAY_SSD1306` | Enable SSD1306 OLED display |
| `DISPLAY_16X2` | Enable 16x2 LCD display |
| `USE_DS18B20` | Enable DS18B20 temperature sensor |
| `USE_DHT` | Enable DHT11/22 sensor |
| `USE_INTERNAL_SENSOR` | Use ESP32's internal temperature sensor |

## Architecture

### File Structure

```
├── ESP_Miner.ino     # Main application code
├── Settings.h        # User configuration and hardware settings
├── MiningJob.h       # Mining job management and SHA1 hashing
├── DSHA1.h           # Optimized SHA1 implementation
├── Counter.h         # High-performance string counter
├── Dashboard.h       # Web dashboard HTML template
└── DisplayHal.h      # Display abstraction layer
```

### Core Components

#### 1. Ring Buffer (Seqlock Implementation)
The miner uses a lock-free ring buffer with canonical seqlock for thread-safe communication between mining tasks and the network submission task.

```
┌─────────────────────────────────────────────────┐
│  Mining Task (Core 0)                           │
│  ├── Computes SHA1 hashes                       │
│  └── Pushes results to ring buffer via ringPush │
└────────────────────┬────────────────────────────┘
                     │ Lock-free seqlock
                     ▼
┌─────────────────────────────────────────────────┐
│  System Task (Core 0)                           │
│  ├── Pops results via ringPop                   │
│  └── Submits shares to pool                     │
└─────────────────────────────────────────────────┘
```

#### 2. PID Temperature Controller
Maintains optimal chip temperature using a proportional-integral-derivative controller:
- **Kp = 0.8**: Proportional gain
- **Ki = 0.05**: Integral gain for steady-state error
- **Kd = 0.2**: Derivative gain for trend prediction
- **Derivative Filter = 0.7**: Smooths temperature fluctuations

#### 3. Task Distribution

| Task | Core | Priority | Purpose |
|------|------|----------|---------|
| System | 0 | 2 | Web server, DNS, WiFi, network |
| Thermal | 0 | 2 | Temperature monitoring & PID control |
| Miner0 | 0 | 3 | Mining on core 0 |
| Miner1 | 1 | 3 | Mining on core 1 |
| Supervisor | 0 | 1 | Task health monitoring |

#### 4. Mining Flow

```
1. SelectNode()          → Fetch best pool from server.duinocoin.com
2. connectToNode()       → TCP connection to mining pool
3. askForJob()           → Request mining job (block hash, expected hash, difficulty)
4. mine()                → SHA1 hash computation loop
5. submit()              → Send solution to pool
6. Update statistics     → Increment counters, update hashrate
```

#### 5. NAT Router Implementation
The ESP32 operates in `WIFI_AP_STA` mode:
- **Station (STA)**: Connects to your home WiFi for internet
- **Access Point (AP)**: Creates hotspot for other devices

NAT translation using ESP-IDF's `ip_napt` allows devices connected to the AP to access the internet through the ESP32.

### Key Algorithms

#### Adaptive Difficulty
```cpp
float acceptRate = (float)accepted / shares * 100;
if (acceptRate < 50) return base * 0.5;   // Too many rejects
if (acceptRate < 70) return base * 0.7;
if (acceptRate > 95) return base * 1.1;   // Can handle more
```

#### Dynamic Batch Size
```cpp
if (ping < 50ms)  return 15;  // Low latency: large batches
if (ping < 100ms) return 10;
if (ping < 200ms) return 7;
return 5;                      // High latency: smaller batches
```

#### Node Quality Scoring
Switches nodes when:
- Average ping > 400ms
- Reject count > 5 shares
- Single ping > 800ms

## Web Interface

Access the dashboard at `http://192.168.4.1` (AP mode) or the device's IP address.

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Main dashboard |
| `/scan` | GET | Scan WiFi networks |
| `/save-sta` | POST | Save WiFi credentials |
| `/save-ap` | POST | Save AP settings |
| `/mining` | GET | JSON mining statistics |
| `/reset` | GET | Factory reset |
| `/reboot` | GET | Reboot device |

### JSON API Response (`/mining`)
```json
{
  "node": "node-name",
  "hashrate": 45.67,
  "shares": 1234,
  "accepted": 1200,
  "total_shares": 50000,
  "total_accepted": 49500,
  "ping": 45,
  "temp": 62.5,
  "intensity": 95,
  "quality": 80,
  "ring": 3,
  "drops": 0,
  "adaptive_diff": 100
}
```

## Troubleshooting

### Common Issues

| Problem | Solution |
|---------|----------|
| Won't connect to WiFi | Check credentials in Settings.h or use web interface |
| Low hashrate | Ensure good WiFi signal, check temperature throttling |
| Frequent disconnects | Try a different node, check WiFi interference |
| "Miner task stuck" messages | Normal recovery mechanism, should auto-restart |
| High temperature warnings | Improve ventilation, reduce ambient temperature |

### Serial Monitor Output
Enable `SERIAL_PRINTING` in Settings.h and set baud rate to 500000.

## Performance Tips

1. **Use 240MHz CPU**: Set in `setup()` with `setCpuFrequencyMhz(240)`
2. **Good WiFi Signal**: Position ESP32 for best reception (quality shown in dashboard)
3. **Keep Cool**: Ensure adequate ventilation; thermal throttling reduces performance
4. **Stable Power**: Use quality USB power supply (500mA+ recommended)
5. **Latest Firmware**: Use OTA updates to get performance improvements

## Technical Specifications

- **SHA1 Implementation**: Optimized 80-round SHA1 with compile-time unrolling
- **Ring Buffer Size**: 64 entries with seqlock synchronization
- **Watchdog Timeout**: 20 seconds
- **Thermal Target**: 65°C (PID-controlled)
- **DNS Cache TTL**: 10 minutes
- **HTTP Buffer**: 2KB
- **Task Stack Size**: 16KB per mining task

## License

Based on the official [Duino-Coin ESP32 Miner](https://github.com/revoxhere/duino-coin).

## Credits

- [Duino-Coin Team](https://duinocoin.com) - Original miner implementation
- ESP-IDF and Arduino-ESP32 communities
- Contributors to the optimization improvements

## Disclaimer

Cryptocurrency mining may consume significant power and generate heat. Ensure proper ventilation and monitor device temperature. Use at your own risk.
