## 6. DEPLOYMENT & TESTING LENGKAP

### 6.1 Setup Raspberry Pi (Detail)

```bash
# 1. Update sistem dan install dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv sqlite3 git curl -y

# 2. Set static IP untuk Raspberry Pi
sudo nano /etc/dhcpcd.conf

# Tambahkan di akhir file:
interface wlan0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 8.8.8.8

# 3. Restart networking
sudo systemctl daemon-reload
sudo systemctl restart dhcpcd

# 4. Verifikasi IP address
ip addr show wlan0

# 5. Setup project directory
mkdir ~/iot_gateway
cd ~/iot_gateway

# 6. Create virtual environment
python3 -m venv venv
source venv/bin/activate

# 7. Install Python packages
pip install flask flask-cors sqlite3 requests python-dateutil

# 8. Create database initialization script
cat > init_db.py << 'EOF'
import sqlite3

def init_database():
    conn = sqlite3.connect('iot_data.db')
    cursor = conn.cursor()
    
    # Drop tables if exist (for fresh start)
    cursor.execute('DROP TABLE IF EXISTS sensor_data')
    cursor.execute('DROP TABLE IF EXISTS node_controls')
    
    # Create sensor_data table
    cursor.execute('''
        CREATE TABLE sensor_data (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            node_id INTEGER NOT NULL,
            node_type TEXT NOT NULL,
            temperature REAL,
            humidity REAL,
            tds REAL,
            relay_state BOOLEAN,
            manual_mode BOOLEAN,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Create node_controls table
    cursor.execute('''
        CREATE TABLE node_controls (
            node_id INTEGER PRIMARY KEY,
            manual_mode BOOLEAN DEFAULT FALSE,
            relay_command BOOLEAN DEFAULT FALSE,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Insert default control states for all 8 nodes
    for node_id in range(1, 9):
        cursor.execute('''
            INSERT INTO node_controls (node_id, manual_mode, relay_command) 
            VALUES (?, FALSE, FALSE)
        ''', (node_id,))
    
    conn.commit()
    conn.close()
    print("Database initialized successfully!")

if __name__ == "__main__":
    init_database()
EOF

# 9. Initialize database
python init_db.py

# 10. Test database
sqlite3 iot_data.db "SELECT * FROM node_controls;"
```

### 6.2 Setup ESP32 Programming Environment

```bash
# Install Arduino IDE di laptop
# Download dari: https://www.arduino.cc/en/software

# Setelah install Arduino IDE:
# 1. Buka Arduino IDE
# 2. File -> Preferences
# 3. Additional Board Manager URLs: 
#    https://dl.espressif.com/dl/package_esp32_index.json
# 4. Tools -> Board -> Board Manager
# 5. Search "ESP32" dan install "ESP32 by Espressif Systems"
# 6. Tools -> Manage Libraries
# 7. Install libraries berikut:
#    - DHT sensor library by Adafruit
#    - ArduinoJson by Benoit Blanchon
#    - Adafruit Unified Sensor
```

### 6.3 Network Configuration & Testing

```bash
# Test koneksi dari laptop ke Raspberry Pi
ping 192.168.1.100

# Test API endpoints dari laptop
curl -X GET http://192.168.1.100:5000/health

# Test dengan data dummy
curl -X POST http://192.168.1.100:5000/api/data \
  -H "Content-Type: application/json" \
  -d '{
    "node_id": 1,
    "node_type": "exhaust",
    "temperature": 25.5,
    "humidity": 60.0,
    "relay_state": false,
    "manual_mode": false,
    "timestamp": 123456789
  }'

# Verify data masuk ke database
sqlite3 ~/iot_gateway/iot_data.db "SELECT * FROM sensor_data ORDER BY timestamp DESC LIMIT 5;"

# Test control endpoint
curl -X GET http://192.168.1.100:5000/api/control/1
curl -X POST http://192.168.1.100:5000/api/control/1 \
  -H "Content-Type: application/json" \
  -d '{"manual_mode": true, "relay_command": true}'
```

### 6.4 ESP32 Hardware Wiring

#### Node Exhaust (DHT22 + Relay):
```
ESP32 Pin -> Component
====================
Pin 4      -> DHT22 Data
Pin 2      -> Relay IN
3.3V       -> DHT22 VCC, Relay VCC
GND        -> DHT22 GND, Relay GND
```

#### Node Pompa (TDS + Relay):
```
ESP32 Pin -> Component
====================
Pin A0     -> TDS Sensor Analog Out
Pin 2      -> Relay IN
3.3V       -> TDS VCC, Relay VCC
GND        -> TDS GND, Relay GND
```

### 6.5 ESP32 Programming & Flashing

```cpp
// Konfigurasi WiFi untuk setiap ESP32
// Node 1 (Exhaust): NODE_ID = 1
// Node 2 (Exhaust): NODE_ID = 2  
// Node 3 (Exhaust): NODE_ID = 3
// Node 4 (Exhaust): NODE_ID = 4
// Node 5 (Pompa): NODE_ID = 5
// Node 6 (Pom# Proyek IoT Terdistribusi ESP32-Raspberry Pi
## Arsitektur Sistem Lengkap

### 📋 Daftar Komponen
- **8 ESP32 Nodes:**
  - 4 Node Exhaust (DHT22 + Relay)
  - 4 Node Pompa (TDS + Relay)
- **1 Raspberry Pi 4** (Gateway Server)
- **Web Monitoring** (React - di laptop)
- **Mobile App** (Flutter - di laptop)

### 🌐 Topologi Jaringan
```
[ESP32 Nodes] ↔ [Wi-Fi Router] ↔ [Raspberry Pi Gateway] ↔ [Laptop (Web/Mobile)]
     HTTP              LAN             HTTP API              HTTP
```

## 1. KODE ESP32

### 1.1 Node Exhaust (DHT22 + Relay)

```cpp
// exhaust_node.ino
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <DHT.h>

// Network Configuration
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";
const char* server_ip = "192.168.1.100"; // Raspberry Pi IP
const int server_port = 5000;

// Hardware Configuration
#define DHT_PIN 4
#define RELAY_PIN 2
#define DHT_TYPE DHT22
#define NODE_ID 1 // Ubah untuk setiap node (1-4)

// Sensor & Control
DHT dht(DHT_PIN, DHT_TYPE);
bool relay_state = false;
bool manual_mode = false;
float temp_threshold = 30.0; // Celsius

// Timing
unsigned long last_sensor_read = 0;
unsigned long last_data_send = 0;
unsigned long last_control_check = 0;
const unsigned long SENSOR_INTERVAL = 2000;
const unsigned long SEND_INTERVAL = 10000;
const unsigned long CONTROL_INTERVAL = 5000;

void setup() {
  Serial.begin(115200);
  
  // Initialize pins
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  // Initialize DHT
  dht.begin();
  
  // Connect WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("WiFi connected!");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  unsigned long current_time = millis();
  
  // Read sensors
  if (current_time - last_sensor_read >= SENSOR_INTERVAL) {
    readSensors();
    last_sensor_read = current_time;
  }
  
  // Send data to server
  if (current_time - last_data_send >= SEND_INTERVAL) {
    sendSensorData();
    last_data_send = current_time;
  }
  
  // Check control commands
  if (current_time - last_control_check >= CONTROL_INTERVAL) {
    checkControlCommands();
    last_control_check = current_time;
  }
  
  // Auto control (if not in manual mode)
  if (!manual_mode) {
    autoControlExhaust();
  }
  
  delay(100);
}

void readSensors() {
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();
  
  if (!isnan(temperature) && !isnan(humidity)) {
    Serial.printf("Temp: %.2f°C, Humidity: %.2f%%\n", temperature, humidity);
  }
}

void sendSensorData() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String("http://") + server_ip + ":" + server_port + "/api/data");
    http.addHeader("Content-Type", "application/json");
    
    // Create JSON payload
    DynamicJsonDocument doc(1024);
    doc["node_id"] = NODE_ID;
    doc["node_type"] = "exhaust";
    doc["temperature"] = dht.readTemperature();
    doc["humidity"] = dht.readHumidity();
    doc["relay_state"] = relay_state;
    doc["manual_mode"] = manual_mode;
    doc["timestamp"] = millis();
    
    String json_string;
    serializeJson(doc, json_string);
    
    int response_code = http.POST(json_string);
    if (response_code > 0) {
      Serial.printf("Data sent successfully: %d\n", response_code);
    } else {
      Serial.printf("Error sending data: %d\n", response_code);
    }
    
    http.end();
  }
}

void checkControlCommands() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String("http://") + server_ip + ":" + server_port + "/api/control/" + NODE_ID);
    
    int response_code = http.GET();
    if (response_code == 200) {
      String payload = http.getString();
      
      DynamicJsonDocument doc(1024);
      deserializeJson(doc, payload);
      
      bool new_manual_mode = doc["manual_mode"];
      bool new_relay_command = doc["relay_command"];
      
      if (new_manual_mode != manual_mode) {
        manual_mode = new_manual_mode;
        Serial.printf("Manual mode: %s\n", manual_mode ? "ON" : "OFF");
      }
      
      if (manual_mode && new_relay_command != relay_state) {
        setRelay(new_relay_command);
        Serial.printf("Manual relay command: %s\n", new_relay_command ? "ON" : "OFF");
      }
    }
    
    http.end();
  }
}

void autoControlExhaust() {
  float temperature = dht.readTemperature();
  
  if (!isnan(temperature)) {
    if (temperature > temp_threshold && !relay_state) {
      setRelay(true);
      Serial.println("Auto: Exhaust ON (temperature high)");
    } else if (temperature < (temp_threshold - 2.0) && relay_state) {
      setRelay(false);
      Serial.println("Auto: Exhaust OFF (temperature normal)");
    }
  }
}

void setRelay(bool state) {
  relay_state = state;
  digitalWrite(RELAY_PIN, state ? HIGH : LOW);
}
```

### 1.2 Node Pompa (TDS + Relay)

```cpp
// pump_node.ino
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

// Network Configuration
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";
const char* server_ip = "192.168.1.100";
const int server_port = 5000;

// Hardware Configuration
#define TDS_PIN A0
#define RELAY_PIN 2
#define NODE_ID 5 // Ubah untuk setiap node (5-8)

// TDS Calibration
const float VREF = 3.3;
const int ADC_RESOLUTION = 4096;
const float TDS_FACTOR = 0.5; // Calibration factor

// Control variables
bool relay_state = false;
bool manual_mode = false;
float tds_threshold = 800.0; // ppm
unsigned long pump_start_time = 0;
const unsigned long PUMP_DURATION = 5000; // 5 seconds

// Timing
unsigned long last_sensor_read = 0;
unsigned long last_data_send = 0;
unsigned long last_control_check = 0;
const unsigned long SENSOR_INTERVAL = 2000;
const unsigned long SEND_INTERVAL = 10000;
const unsigned long CONTROL_INTERVAL = 5000;

void setup() {
  Serial.begin(115200);
  
  // Initialize pins
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(TDS_PIN, INPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  // Connect WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("WiFi connected!");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  unsigned long current_time = millis();
  
  // Handle pump duration
  if (relay_state && !manual_mode && (current_time - pump_start_time >= PUMP_DURATION)) {
    setRelay(false);
    Serial.println("Auto pump cycle completed");
  }
  
  // Read sensors
  if (current_time - last_sensor_read >= SENSOR_INTERVAL) {
    readSensors();
    last_sensor_read = current_time;
  }
  
  // Send data to server
  if (current_time - last_data_send >= SEND_INTERVAL) {
    sendSensorData();
    last_data_send = current_time;
  }
  
  // Check control commands
  if (current_time - last_control_check >= CONTROL_INTERVAL) {
    checkControlCommands();
    last_control_check = current_time;
  }
  
  // Auto control (if not in manual mode)
  if (!manual_mode) {
    autoControlPump();
  }
  
  delay(100);
}

float readTDS() {
  int analog_value = analogRead(TDS_PIN);
  float voltage = (analog_value * VREF) / ADC_RESOLUTION;
  float tds_value = (voltage * 1000) * TDS_FACTOR;
  return tds_value;
}

void readSensors() {
  float tds = readTDS();
  Serial.printf("TDS: %.2f ppm\n", tds);
}

void sendSensorData() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String("http://") + server_ip + ":" + server_port + "/api/data");
    http.addHeader("Content-Type", "application/json");
    
    DynamicJsonDocument doc(1024);
    doc["node_id"] = NODE_ID;
    doc["node_type"] = "pump";
    doc["tds"] = readTDS();
    doc["relay_state"] = relay_state;
    doc["manual_mode"] = manual_mode;
    doc["timestamp"] = millis();
    
    String json_string;
    serializeJson(doc, json_string);
    
    int response_code = http.POST(json_string);
    if (response_code > 0) {
      Serial.printf("Data sent successfully: %d\n", response_code);
    } else {
      Serial.printf("Error sending data: %d\n", response_code);
    }
    
    http.end();
  }
}

void checkControlCommands() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String("http://") + server_ip + ":" + server_port + "/api/control/" + NODE_ID);
    
    int response_code = http.GET();
    if (response_code == 200) {
      String payload = http.getString();
      
      DynamicJsonDocument doc(1024);
      deserializeJson(doc, payload);
      
      bool new_manual_mode = doc["manual_mode"];
      bool new_relay_command = doc["relay_command"];
      
      if (new_manual_mode != manual_mode) {
        manual_mode = new_manual_mode;
        Serial.printf("Manual mode: %s\n", manual_mode ? "ON" : "OFF");
      }
      
      if (manual_mode && new_relay_command != relay_state) {
        setRelay(new_relay_command);
        Serial.printf("Manual relay command: %s\n", new_relay_command ? "ON" : "OFF");
      }
    }
    
    http.end();
  }
}

void autoControlPump() {
  float tds = readTDS();
  
  if (tds < tds_threshold && !relay_state) {
    setRelay(true);
    pump_start_time = millis();
    Serial.println("Auto: Pump ON (low TDS)");
  }
}

void setRelay(bool state) {
  relay_state = state;
  digitalWrite(RELAY_PIN, state ? HIGH : LOW);
}
```

## 2. RASPBERRY PI SETUP

### 2.1 System Setup

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install Python dan dependencies
sudo apt install python3-pip python3-venv sqlite3 -y

# Buat virtual environment
mkdir ~/iot_gateway
cd ~/iot_gateway
python3 -m venv venv
source venv/bin/activate

# Install Python packages
pip install flask flask-cors sqlite3 requests
```

### 2.2 Database Schema (SQLite)

```sql
-- database_setup.sql
CREATE TABLE IF NOT EXISTS sensor_data (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    node_id INTEGER NOT NULL,
    node_type TEXT NOT NULL,
    temperature REAL,
    humidity REAL,
    tds REAL,
    relay_state BOOLEAN,
    manual_mode BOOLEAN,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS node_controls (
    node_id INTEGER PRIMARY KEY,
    manual_mode BOOLEAN DEFAULT FALSE,
    relay_command BOOLEAN DEFAULT FALSE,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insert default control states for all nodes
INSERT OR IGNORE INTO node_controls (node_id, manual_mode, relay_command) VALUES
(1, FALSE, FALSE), (2, FALSE, FALSE), (3, FALSE, FALSE), (4, FALSE, FALSE),
(5, FALSE, FALSE), (6, FALSE, FALSE), (7, FALSE, FALSE), (8, FALSE, FALSE);
```

### 2.3 Flask API Server

```python
# app.py
from flask import Flask, request, jsonify
from flask_cors import CORS
import sqlite3
import json
from datetime import datetime, timedelta
import threading
import time

app = Flask(__name__)
CORS(app)  # Enable CORS for web/mobile access

DATABASE = 'iot_data.db'

def init_database():
    """Initialize database with tables"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    
    # Create tables
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS sensor_data (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            node_id INTEGER NOT NULL,
            node_type TEXT NOT NULL,
            temperature REAL,
            humidity REAL,
            tds REAL,
            relay_state BOOLEAN,
            manual_mode BOOLEAN,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS node_controls (
            node_id INTEGER PRIMARY KEY,
            manual_mode BOOLEAN DEFAULT FALSE,
            relay_command BOOLEAN DEFAULT FALSE,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Insert default control states
    for node_id in range(1, 9):
        cursor.execute('''
            INSERT OR IGNORE INTO node_controls (node_id, manual_mode, relay_command) 
            VALUES (?, FALSE, FALSE)
        ''', (node_id,))
    
    conn.commit()
    conn.close()

def get_db_connection():
    """Get database connection"""
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn

@app.route('/api/data', methods=['POST'])
def receive_sensor_data():
    """Receive sensor data from ESP32 nodes"""
    try:
        data = request.get_json()
        
        conn = get_db_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            INSERT INTO sensor_data 
            (node_id, node_type, temperature, humidity, tds, relay_state, manual_mode)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', (
            data.get('node_id'),
            data.get('node_type'),
            data.get('temperature'),
            data.get('humidity'),
            data.get('tds'),
            data.get('relay_state'),
            data.get('manual_mode')
        ))
        
        conn.commit()
        conn.close()
        
        return jsonify({'status': 'success', 'message': 'Data received'}), 200
        
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

@app.route('/api/control/<int:node_id>', methods=['GET'])
def get_control_commands(node_id):
    """Send control commands to ESP32 nodes"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT manual_mode, relay_command 
            FROM node_controls 
            WHERE node_id = ?
        ''', (node_id,))
        
        result = cursor.fetchone()
        conn.close()
        
        if result:
            return jsonify({
                'manual_mode': bool(result['manual_mode']),
                'relay_command': bool(result['relay_command'])
            }), 200
        else:
            return jsonify({'manual_mode': False, 'relay_command': False}), 200
            
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

@app.route('/api/nodes', methods=['GET'])
def get_all_nodes():
    """Get current status of all nodes"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # Get latest data for each node
        cursor.execute('''
            SELECT s1.* FROM sensor_data s1
            INNER JOIN (
                SELECT node_id, MAX(timestamp) as max_timestamp
                FROM sensor_data
                GROUP BY node_id
            ) s2 ON s1.node_id = s2.node_id AND s1.timestamp = s2.max_timestamp
            ORDER BY s1.node_id
        ''')
        
        nodes = []
        for row in cursor.fetchall():
            nodes.append({
                'node_id': row['node_id'],
                'node_type': row['node_type'],
                'temperature': row['temperature'],
                'humidity': row['humidity'],
                'tds': row['tds'],
                'relay_state': bool(row['relay_state']),
                'manual_mode': bool(row['manual_mode']),
                'timestamp': row['timestamp']
            })
        
        conn.close()
        return jsonify(nodes), 200
        
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

@app.route('/api/control/<int:node_id>', methods=['POST'])
def set_control_commands(node_id):
    """Set control commands for specific node (from web/mobile)"""
    try:
        data = request.get_json()
        
        conn = get_db_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            UPDATE node_controls 
            SET manual_mode = ?, relay_command = ?, updated_at = CURRENT_TIMESTAMP
            WHERE node_id = ?
        ''', (
            data.get('manual_mode', False),
            data.get('relay_command', False),
            node_id
        ))
        
        conn.commit()
        conn.close()
        
        return jsonify({'status': 'success', 'message': 'Control updated'}), 200
        
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

@app.route('/api/history/<int:node_id>', methods=['GET'])
def get_node_history(node_id):
    """Get historical data for specific node"""
    try:
        # Get time range from query parameters
        hours = request.args.get('hours', 24, type=int)
        
        conn = get_db_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT * FROM sensor_data 
            WHERE node_id = ? AND timestamp >= datetime('now', '-{} hours')
            ORDER BY timestamp DESC
            LIMIT 1000
        '''.format(hours), (node_id,))
        
        history = []
        for row in cursor.fetchall():
            history.append({
                'id': row['id'],
                'node_id': row['node_id'],
                'node_type': row['node_type'],
                'temperature': row['temperature'],
                'humidity': row['humidity'],
                'tds': row['tds'],
                'relay_state': bool(row['relay_state']),
                'timestamp': row['timestamp']
            })
        
        conn.close()
        return jsonify(history), 200
        
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

@app.route('/api/dashboard', methods=['GET'])
def get_dashboard_data():
    """Get dashboard summary data"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # Get latest data for all nodes
        cursor.execute('''
            SELECT s1.* FROM sensor_data s1
            INNER JOIN (
                SELECT node_id, MAX(timestamp) as max_timestamp
                FROM sensor_data
                GROUP BY node_id
            ) s2 ON s1.node_id = s2.node_id AND s1.timestamp = s2.max_timestamp
            ORDER BY s1.node_id
        ''')
        
        nodes = cursor.fetchall()
        
        # Calculate summary statistics
        total_nodes = len(nodes)
        active_relays = sum(1 for node in nodes if node['relay_state'])
        manual_nodes = sum(1 for node in nodes if node['manual_mode'])
        
        # Get average values
        exhaust_nodes = [node for node in nodes if node['node_type'] == 'exhaust']
        pump_nodes = [node for node in nodes if node['node_type'] == 'pump']
        
        avg_temp = sum(node['temperature'] or 0 for node in exhaust_nodes) / len(exhaust_nodes) if exhaust_nodes else 0
        avg_humidity = sum(node['humidity'] or 0 for node in exhaust_nodes) / len(exhaust_nodes) if exhaust_nodes else 0
        avg_tds = sum(node['tds'] or 0 for node in pump_nodes) / len(pump_nodes) if pump_nodes else 0
        
        conn.close()
        
        return jsonify({
            'summary': {
                'total_nodes': total_nodes,
                'active_relays': active_relays,
                'manual_nodes': manual_nodes,
                'avg_temperature': round(avg_temp, 2),
                'avg_humidity': round(avg_humidity, 2),
                'avg_tds': round(avg_tds, 2)
            },
            'nodes': [{
                'node_id': node['node_id'],
                'node_type': node['node_type'],
                'temperature': node['temperature'],
                'humidity': node['humidity'],
                'tds': node['tds'],
                'relay_state': bool(node['relay_state']),
                'manual_mode': bool(node['manual_mode']),
                'timestamp': node['timestamp']
            } for node in nodes]
        }), 200
        
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    return jsonify({'status': 'healthy', 'timestamp': datetime.now().isoformat()}), 200

if __name__ == '__main__':
    init_database()
    print("IoT Gateway Server starting...")
    print("Database initialized")
    print("Server running at http://0.0.0.0:5000")
    app.run(host='0.0.0.0', port=5000, debug=False)
```

### 2.4 Service Setup (Raspberry Pi)

```bash
# Buat systemd service
sudo nano /etc/systemd/system/iot-gateway.service
```

```ini
# /etc/systemd/system/iot-gateway.service
[Unit]
Description=IoT Gateway Server
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/iot_gateway
Environment=PATH=/home/pi/iot_gateway/venv/bin
ExecStart=/home/pi/iot_gateway/venv/bin/python app.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Enable dan start service
sudo systemctl daemon-reload
sudo systemctl enable iot-gateway.service
sudo systemctl start iot-gateway.service

# Check status
sudo systemctl status iot-gateway.service
```

## 3. WEB MONITORING (React)

### 3.1 Project Setup

```bash
# Buat React project
npx create-react-app iot-web-monitor
cd iot-web-monitor

# Install dependencies
npm install axios recharts lucide-react tailwindcss
npm install -D @tailwindcss/forms

# Initialize Tailwind CSS
npx tailwindcss init -p
```

### 3.2 Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [
    require('@tailwindcss/forms'),
  ],
}
```

```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 3.3 React Components

Karena kode React akan sangat panjang, saya akan buat dalam artifact terpisah untuk web monitoring interface.

## 4. INSTALASI & PENGUJIAN

### 4.1 Network Configuration

```bash
# Raspberry Pi static IP
sudo nano /etc/dhcpcd.conf

# Tambahkan:
interface wlan0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1

# Restart networking
sudo systemctl daemon-reload
sudo systemctl restart dhcpcd
```

### 4.2 ESP32 Flash Instructions

1. Install Arduino IDE
2. Install ESP32 Board Package
3. Install libraries: WiFi, HTTPClient, ArduinoJson, DHT sensor library
4. Set Board: "ESP32 Dev Module"
5. Upload kode ke masing-masing ESP32

### 4.3 Testing Procedure

```bash
# Test Raspberry Pi API
curl -X GET http://192.168.1.100:5000/health

# Test node data submission
curl -X POST http://192.168.1.100:5000/api/data \
  -H "Content-Type: application/json" \
  -d '{"node_id":1,"node_type":"exhaust","temperature":25.5,"humidity":60.0,"relay_state":false,"manual_mode":false}'

# Test control endpoint
curl -X GET http://192.168.1.100:5000/api/control/1
```

### 4.4 Troubleshooting Common Issues

1. **ESP32 tidak konek WiFi**: Periksa SSID/password, jarak ke router
2. **Data tidak masuk database**: Cek endpoint URL, firewall Raspberry Pi
3. **Relay tidak berfungsi**: Periksa koneksi pin, tegangan relay
4. **Sensor TDS tidak akurat**: Kalibrasi sensor dengan larutan standar
5. **Web tidak bisa akses**: Periksa CORS settings, network connectivity

## 5. MONITORING & MAINTENANCE

### 5.1 Log Monitoring

```bash
# View service logs
sudo journalctl -u iot-gateway.service -f

# View database size
ls -lh ~/iot_gateway/iot_data.db

# Monitor system resources
htop
```

### 5.2 Data Backup

```bash
# Backup database
cp ~/iot_gateway/iot_data.db ~/backup/iot_data_$(date +%Y%m%d).db

# Auto backup script (crontab)
0 2 * * * cp ~/iot_gateway/iot_data.db ~/backup/iot_data_$(date +\%Y\%m\%d).db
```

### 5.3 Performance Optimization

- Bersihkan data lama secara berkala (>30 hari)
- Monitor penggunaan CPU/Memory Raspberry Pi
- Optimalkan interval pengiriman data ESP32
- Gunakan connection pooling untuk database

Ini adalah framework lengkap untuk proyek IoT Anda. Selanjutnya saya akan buat komponen web monitoring dan mobile app dalam artifact terpisah.
