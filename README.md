# Rencana Proyek IoT Hidroponik - Arsitektur Terdistribusi

## 1. Desain Sistem dan Topologi

### 1.1 Arsitektur Sistem
```
┌─────────────────────────────────────────────────────────────┐
│                     LAPTOP (Monitoring)                    │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   Web Monitor   │    │      Mobile App                 │ │
│  │   (React.js)    │    │   (React Native)                │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │ HTTP/REST API
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  RASPBERRY PI (Gateway)                    │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   REST API      │    │      Database                   │ │
│  │   (Flask/Node)  │    │   (PostgreSQL)                  │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │ Wi-Fi Network
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    ESP32 NODES (8 Units)                   │
│                                                             │
│  EXHAUST CONTROL (4 Nodes)    NUTRIENT CONTROL (4 Nodes)   │
│  ┌─────────────────────┐      ┌─────────────────────────┐   │
│  │ ESP32 + DHT22       │      │ ESP32 + TDS Sensor      │   │
│  │ + Relay + Fan       │      │ + Relay + Pump          │   │
│  └─────────────────────┘      └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Wiring Diagram per Node

#### Node Exhaust (ESP32 + DHT22 + Relay)
```
ESP32 GPIO Pins:
- GPIO 4  → DHT22 Data Pin
- GPIO 2  → Relay IN (untuk Exhaust Fan)
- 3.3V    → DHT22 VCC, Relay VCC
- GND     → DHT22 GND, Relay GND

Relay Connection:
- Relay COM → AC Line (Live) ke Exhaust Fan
- Relay NO  → Exhaust Fan AC Input
- AC Neutral → Langsung ke Exhaust Fan
```

#### Node Nutrient (ESP32 + TDS + Relay)
```
ESP32 GPIO Pins:
- GPIO 34 → TDS Sensor Analog Out (ADC)
- GPIO 2  → Relay IN (untuk Pompa)
- 3.3V    → TDS Sensor VCC, Relay VCC
- GND     → TDS Sensor GND, Relay GND

TDS Sensor:
- VCC → 3.3V atau 5V
- GND → Ground
- A   → GPIO 34 (ADC Pin)

Relay Connection:
- Relay COM → DC 12V+ ke Pompa
- Relay NO  → Pompa DC Input+
- DC 12V-   → Langsung ke Pompa DC Input-
```

## 2. Pemrograman ESP32

### 2.1 Kode untuk Node Exhaust Control

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <DHT.h>
#include <ArduinoJson.h>

// WiFi credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Raspberry Pi server
const char* serverURL = "http://192.168.1.100:5000"; // IP Raspberry Pi

// DHT22 setup
#define DHT_PIN 4
#define DHT_TYPE DHT22
DHT dht(DHT_PIN, DHT_TYPE);

// Relay setup
#define RELAY_PIN 2

// Node configuration
int nodeId = 1; // Ganti untuk setiap node (1-4)
float tempThreshold = 30.0; // Suhu ambang batas
bool manualControl = false;
bool relayState = false;

void setup() {
  Serial.begin(115200);
  
  // Initialize pins
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  // Initialize DHT
  dht.begin();
  
  // Connect to WiFi
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
  // Read sensors
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();
  
  if (isnan(temperature) || isnan(humidity)) {
    Serial.println("Failed to read from DHT sensor!");
    delay(5000);
    return;
  }
  
  // Check for manual control commands
  checkManualControl();
  
  // Automatic control logic
  if (!manualControl) {
    if (temperature > tempThreshold && !relayState) {
      digitalWrite(RELAY_PIN, HIGH);
      relayState = true;
      Serial.println("Exhaust ON - Temperature high");
    } else if (temperature <= (tempThreshold - 2) && relayState) {
      digitalWrite(RELAY_PIN, LOW);
      relayState = false;
      Serial.println("Exhaust OFF - Temperature normal");
    }
  }
  
  // Send data to server
  sendDataToServer(temperature, humidity, relayState);
  
  delay(10000); // Send data every 10 seconds
}

void sendDataToServer(float temp, float hum, bool relay) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String(serverURL) + "/api/exhaust/data");
    http.addHeader("Content-Type", "application/json");
    
    // Create JSON payload
    StaticJsonDocument<200> doc;
    doc["nodeId"] = nodeId;
    doc["temperature"] = temp;
    doc["humidity"] = hum;
    doc["relayState"] = relay;
    doc["timestamp"] = millis();
    
    String jsonString;
    serializeJson(doc, jsonString);
    
    int httpResponseCode = http.POST(jsonString);
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      Serial.println("Data sent successfully");
    } else {
      Serial.println("Error sending data");
    }
    
    http.end();
  }
}

void checkManualControl() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String(serverURL) + "/api/exhaust/control/" + String(nodeId));
    
    int httpResponseCode = http.GET();
    
    if (httpResponseCode > 0) {
      String payload = http.getString();
      
      StaticJsonDocument<200> doc;
      deserializeJson(doc, payload);
      
      if (doc.containsKey("manualControl")) {
        manualControl = doc["manualControl"];
        if (manualControl && doc.containsKey("relayState")) {
          bool newState = doc["relayState"];
          digitalWrite(RELAY_PIN, newState ? HIGH : LOW);
          relayState = newState;
          Serial.println("Manual control activated");
        }
      }
    }
    
    http.end();
  }
}
```

### 2.2 Kode untuk Node Nutrient Control

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

// WiFi credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Server configuration
const char* serverURL = "http://192.168.1.100:5000";

// Pin definitions
#define TDS_PIN 34
#define RELAY_PIN 2

// Node configuration
int nodeId = 1; // Ganti untuk setiap node (1-4)
float tdsThresholdMin = 800.0; // TDS minimum (ppm)
float tdsThresholdMax = 1200.0; // TDS maximum (ppm)
bool manualControl = false;
bool relayState = false;

// TDS calibration
float vref = 3.3;
float tdsCalibrationFactor = 0.5;

void setup() {
  Serial.begin(115200);
  
  // Initialize pins
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  // Connect to WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("WiFi connected!");
}

void loop() {
  // Read TDS sensor
  float tdsValue = readTDS();
  
  // Check for manual control
  checkManualControl();
  
  // Automatic control logic
  if (!manualControl) {
    if (tdsValue < tdsThresholdMin && !relayState) {
      digitalWrite(RELAY_PIN, HIGH);
      relayState = true;
      Serial.println("Pump ON - TDS too low");
    } else if (tdsValue >= tdsThresholdMax && relayState) {
      digitalWrite(RELAY_PIN, LOW);
      relayState = false;
      Serial.println("Pump OFF - TDS sufficient");
    }
  }
  
  // Send data to server
  sendDataToServer(tdsValue, relayState);
  
  delay(10000);
}

float readTDS() {
  int analogValue = analogRead(TDS_PIN);
  float voltage = analogValue * (vref / 4095.0);
  float tdsValue = (133.42 * voltage * voltage * voltage 
                   - 255.86 * voltage * voltage 
                   + 857.39 * voltage) * tdsCalibrationFactor;
  
  return tdsValue;
}

void sendDataToServer(float tds, bool relay) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String(serverURL) + "/api/nutrient/data");
    http.addHeader("Content-Type", "application/json");
    
    StaticJsonDocument<200> doc;
    doc["nodeId"] = nodeId;
    doc["tdsValue"] = tds;
    doc["relayState"] = relay;
    doc["timestamp"] = millis();
    
    String jsonString;
    serializeJson(doc, jsonString);
    
    int httpResponseCode = http.POST(jsonString);
    
    if (httpResponseCode > 0) {
      Serial.println("Data sent successfully");
    }
    
    http.end();
  }
}

void checkManualControl() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(String(serverURL) + "/api/nutrient/control/" + String(nodeId));
    
    int httpResponseCode = http.GET();
    
    if (httpResponseCode > 0) {
      String payload = http.getString();
      
      StaticJsonDocument<200> doc;
      deserializeJson(doc, payload);
      
      if (doc.containsKey("manualControl")) {
        manualControl = doc["manualControl"];
        if (manualControl && doc.containsKey("relayState")) {
          bool newState = doc["relayState"];
          digitalWrite(RELAY_PIN, newState ? HIGH : LOW);
          relayState = newState;
        }
      }
    }
    
    http.end();
  }
}
```

## 3. Konfigurasi Raspberry Pi

### 3.1 Instalasi OS dan Dependencies

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install Python dan pip
sudo apt install python3 python3-pip python3-venv -y

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib -y

# Install Node.js (opsional)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y

# Install Git
sudo apt install git -y
```

### 3.2 Setup Database PostgreSQL

```bash
# Masuk ke PostgreSQL
sudo -u postgres psql

# Buat database dan user
CREATE DATABASE hydroponics_iot;
CREATE USER iot_user WITH ENCRYPTED PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE hydroponics_iot TO iot_user;
\q
```

### 3.3 Backend Server (Flask)

```python
# app.py
from flask import Flask, request, jsonify
from flask_cors import CORS
import psycopg2
from datetime import datetime
import json

app = Flask(__name__)
CORS(app)

# Database configuration
DB_CONFIG = {
    'host': 'localhost',
    'database': 'hydroponics_iot',
    'user': 'iot_user',
    'password': 'your_password'
}

# Manual control states
manual_controls = {
    'exhaust': {},
    'nutrient': {}
}

def get_db_connection():
    return psycopg2.connect(**DB_CONFIG)

def init_database():
    conn = get_db_connection()
    cur = conn.cursor()
    
    # Create tables
    cur.execute('''
        CREATE TABLE IF NOT EXISTS exhaust_data (
            id SERIAL PRIMARY KEY,
            node_id INTEGER,
            temperature REAL,
            humidity REAL,
            relay_state BOOLEAN,
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    cur.execute('''
        CREATE TABLE IF NOT EXISTS nutrient_data (
            id SERIAL PRIMARY KEY,
            node_id INTEGER,
            tds_value REAL,
            relay_state BOOLEAN,
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    conn.commit()
    cur.close()
    conn.close()

# API Endpoints for Exhaust Control
@app.route('/api/exhaust/data', methods=['POST'])
def receive_exhaust_data():
    data = request.json
    
    conn = get_db_connection()
    cur = conn.cursor()
    
    cur.execute('''
        INSERT INTO exhaust_data (node_id, temperature, humidity, relay_state)
        VALUES (%s, %s, %s, %s)
    ''', (data['nodeId'], data['temperature'], data['humidity'], data['relayState']))
    
    conn.commit()
    cur.close()
    conn.close()
    
    return jsonify({'status': 'success'})

@app.route('/api/exhaust/control/<int:node_id>', methods=['GET'])
def get_exhaust_control(node_id):
    if node_id in manual_controls['exhaust']:
        return jsonify(manual_controls['exhaust'][node_id])
    return jsonify({'manualControl': False})

@app.route('/api/exhaust/control/<int:node_id>', methods=['POST'])
def set_exhaust_control(node_id):
    data = request.json
    manual_controls['exhaust'][node_id] = data
    return jsonify({'status': 'success'})

# API Endpoints for Nutrient Control
@app.route('/api/nutrient/data', methods=['POST'])
def receive_nutrient_data():
    data = request.json
    
    conn = get_db_connection()
    cur = conn.cursor()
    
    cur.execute('''
        INSERT INTO nutrient_data (node_id, tds_value, relay_state)
        VALUES (%s, %s, %s)
    ''', (data['nodeId'], data['tdsValue'], data['relayState']))
    
    conn.commit()
    cur.close()
    conn.close()
    
    return jsonify({'status': 'success'})

@app.route('/api/nutrient/control/<int:node_id>', methods=['GET'])
def get_nutrient_control(node_id):
    if node_id in manual_controls['nutrient']:
        return jsonify(manual_controls['nutrient'][node_id])
    return jsonify({'manualControl': False})

@app.route('/api/nutrient/control/<int:node_id>', methods=['POST'])
def set_nutrient_control(node_id):
    data = request.json
    manual_controls['nutrient'][node_id] = data
    return jsonify({'status': 'success'})

# API for Web/Mobile monitoring
@app.route('/api/exhaust/latest', methods=['GET'])
def get_latest_exhaust_data():
    conn = get_db_connection()
    cur = conn.cursor()
    
    cur.execute('''
        SELECT DISTINCT ON (node_id) node_id, temperature, humidity, relay_state, timestamp
        FROM exhaust_data
        ORDER BY node_id, timestamp DESC
    ''')
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    data = []
    for row in results:
        data.append({
            'nodeId': row[0],
            'temperature': row[1],
            'humidity': row[2],
            'relayState': row[3],
            'timestamp': row[4].isoformat()
        })
    
    return jsonify(data)

@app.route('/api/nutrient/latest', methods=['GET'])
def get_latest_nutrient_data():
    conn = get_db_connection()
    cur = conn.cursor()
    
    cur.execute('''
        SELECT DISTINCT ON (node_id) node_id, tds_value, relay_state, timestamp
        FROM nutrient_data
        ORDER BY node_id, timestamp DESC
    ''')
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    data = []
    for row in results:
        data.append({
            'nodeId': row[0],
            'tdsValue': row[1],
            'relayState': row[2],
            'timestamp': row[3].isoformat()
        })
    
    return jsonify(data)

@app.route('/api/exhaust/history/<int:node_id>', methods=['GET'])
def get_exhaust_history(node_id):
    hours = request.args.get('hours', 24, type=int)
    
    conn = get_db_connection()
    cur = conn.cursor()
    
    cur.execute('''
        SELECT temperature, humidity, relay_state, timestamp
        FROM exhaust_data
        WHERE node_id = %s AND timestamp > NOW() - INTERVAL '%s hours'
        ORDER BY timestamp DESC
        LIMIT 100
    ''', (node_id, hours))
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    data = []
    for row in results:
        data.append({
            'temperature': row[0],
            'humidity': row[1],
            'relayState': row[2],
            'timestamp': row[3].isoformat()
        })
    
    return jsonify(data)

@app.route('/api/nutrient/history/<int:node_id>', methods=['GET'])
def get_nutrient_history(node_id):
    hours = request.args.get('hours', 24, type=int)
    
    conn = get_db_connection()
    cur = conn.cursor()
    
    cur.execute('''
        SELECT tds_value, relay_state, timestamp
        FROM nutrient_data
        WHERE node_id = %s AND timestamp > NOW() - INTERVAL '%s hours'
        ORDER BY timestamp DESC
        LIMIT 100
    ''', (node_id, hours))
    
    results = cur.fetchall()
    cur.close()
    conn.close()
    
    data = []
    for row in results:
        data.append({
            'tdsValue': row[0],
            'relayState': row[1],
            'timestamp': row[2].isoformat()
        })
    
    return jsonify(data)

if __name__ == '__main__':
    init_database()
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### 3.4 Systemd Service untuk Auto-start

```bash
# Buat file service
sudo nano /etc/systemd/system/hydroponics-api.service

# Isi file service:
[Unit]
Description=Hydroponics IoT API Server
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/hydroponics-iot
Environment=PATH=/home/pi/hydroponics-iot/venv/bin
ExecStart=/home/pi/hydroponics-iot/venv/bin/python app.py
Restart=always

[Install]
WantedBy=multi-user.target

# Enable dan start service
sudo systemctl enable hydroponics-api.service
sudo systemctl start hydroponics-api.service
```

## 4. Web Monitoring Interface

### 4.1 React.js Frontend Setup

```bash
# Buat project React
npx create-react-app hydroponics-web
cd hydroponics-web

# Install dependencies
npm install axios recharts lucide-react
```

### 4.2 Main Dashboard Component

```jsx
// src/App.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';
import './App.css';

const API_BASE_URL = 'http://192.168.1.100:5000'; // IP Raspberry Pi

function App() {
  const [exhaustData, setExhaustData] = useState([]);
  const [nutrientData, setNutrientData] = useState([]);
  const [selectedNode, setSelectedNode] = useState(1);
  const [historyData, setHistoryData] = useState([]);
  const [activeTab, setActiveTab] = useState('exhaust');

  useEffect(() => {
    fetchLatestData();
    const interval = setInterval(fetchLatestData, 10000); // Update setiap 10 detik
    return () => clearInterval(interval);
  }, []);

  useEffect(() => {
    if (selectedNode) {
      fetchHistoryData(selectedNode);
    }
  }, [selectedNode, activeTab]);

  const fetchLatestData = async () => {
    try {
      const [exhaustResponse, nutrientResponse] = await Promise.all([
        axios.get(`${API_BASE_URL}/api/exhaust/latest`),
        axios.get(`${API_BASE_URL}/api/nutrient/latest`)
      ]);
      
      setExhaustData(exhaustResponse.data);
      setNutrientData(nutrientResponse.data);
    } catch (error) {
      console.error('Error fetching latest data:', error);
    }
  };

  const fetchHistoryData = async (nodeId) => {
    try {
      const endpoint = activeTab === 'exhaust' 
        ? `/api/exhaust/history/${nodeId}?hours=24`
        : `/api/nutrient/history/${nodeId}?hours=24`;
      
      const response = await axios.get(`${API_BASE_URL}${endpoint}`);
      setHistoryData(response.data.reverse()); // Reverse untuk chronological order
    } catch (error) {
      console.error('Error fetching history data:', error);
    }
  };

  const toggleRelay = async (nodeId, newState) => {
    try {
      const endpoint = activeTab === 'exhaust' 
        ? `/api/exhaust/control/${nodeId}`
        : `/api/nutrient/control/${nodeId}`;
      
      await axios.post(`${API_BASE_URL}${endpoint}`, {
        manualControl: true,
        relayState: newState
      });
      
      // Refresh data
      fetchLatestData();
    } catch (error) {
      console.error('Error toggling relay:', error);
    }
  };

  const resetToAuto = async (nodeId) => {
    try {
      const endpoint = activeTab === 'exhaust' 
        ? `/api/exhaust/control/${nodeId}`
        : `/api/nutrient/control/${nodeId}`;
      
      await axios.post(`${API_BASE_URL}${endpoint}`, {
        manualControl: false
      });
      
      fetchLatestData();
    } catch (error) {
      console.error('Error resetting to auto:', error);
    }
  };

  const renderNodeCards = () => {
    const data = activeTab === 'exhaust' ? exhaustData : nutrientData;
    
    return data.map((node) => (
      <div key={node.nodeId} className="node-card">
        <h3>Node {node.nodeId}</h3>
        
        {activeTab === 'exhaust' ? (
          <div className="sensor-data">
            <p>Suhu: <span className="value">{node.temperature?.toFixed(1)}°C</span></p>
            <p>Kelembaban: <span className="value">{node.humidity?.toFixed(1)}%</span></p>
          </div>
        ) : (
          <div className="sensor-data">
            <p>TDS: <span className="value">{node.tdsValue?.toFixed(0)} ppm</span></p>
          </div>
        )}
        
        <div className="control-section">
          <p>Status: <span className={`status ${node.relayState ? 'on' : 'off'}`}>
            {node.relayState ? 'ON' : 'OFF'}
          </span></p>
          
          <div className="control-buttons">
            <button 
              onClick={() => toggleRelay(node.nodeId, true)}
              disabled={node.relayState}
              className="btn btn-on"
            >
              Turn ON
            </button>
            <button 
              onClick={() => toggleRelay(node.nodeId, false)}
              disabled={!node.relayState}
              className="btn btn-off"
            >
              Turn OFF
            </button>
            <button 
              onClick={() => resetToAuto(node.nodeId)}
              className="btn btn-auto"
            >
              Auto Mode
            </button>
          </div>
        </div>
        
        <button 
          onClick={() => setSelectedNode(node.nodeId)}
          className={`btn btn-chart ${selectedNode === node.nodeId ? 'active' : ''}`}
        >
          View Chart
        </button>
      </div>
    ));
  };

  const renderChart = () => {
    if (!historyData.length) return <p>Loading chart data...</p>;

    return (
      <div className="chart-container">
        <h3>Node {selectedNode} - Historical Data (24 hours)</h3>
        <ResponsiveContainer width="100%" height={400}>
          <LineChart data={historyData}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis 
              dataKey="timestamp" 
              tickFormatter={(value) => new Date(value).toLocaleTimeString()}
            />
            <YAxis />
            <Tooltip 
              labelFormatter={(value) => new Date(value).toLocaleString()}
            />
            <Legend />
            
            {activeTab === 'exhaust' ? (
              <>
                <Line type="monotone" dataKey="temperature" stroke="#ff7300" name="Temperature (°C)" />
                <Line type="monotone" dataKey="humidity" stroke="#387908" name="Humidity (%)" />
              </>
            ) : (
              <Line type="monotone" dataKey="tdsValue" stroke="#8884d8" name="TDS (ppm)" />
            )}
          </LineChart>
        </ResponsiveContainer>
      </div>
    );
  };

  return (
    <div className="App">
      <header>
        <h1>Hydroponics IoT Monitoring System</h1>
        <div className="tab-buttons">
          <button 
            onClick={() => setActiveTab('exhaust')}
            className={`tab-btn ${activeTab === 'exhaust' ? 'active' : ''}`}
          >
            Exhaust Control
          </button>
          <button 
            onClick={() => setActiveTab('nutrient')}
            className={`tab-btn ${activeTab === 'nutrient' ? 'active' : ''}`}
          >
            Nutrient Control
          </button>
        </div>
      </header>

      <main>
        <div className="nodes-grid">
          {renderNodeCards()}
        </div>
        
        <div className="chart-section">
          {renderChart()}
        </div>
      </main>
    </div>
  );
}

export default App;
```

### 4.3 CSS Styling

```css
/* src/App.css */
.App {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  margin: 0;
  padding: 20px;
  background-color: #f5f5f5;
}

header {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  margin-bottom: 20px;
}

header h1 {
  margin: 0 0 20px 0;
  color: #333;
}

.tab-buttons {
  display: flex;
  gap: 10px;
}

.tab-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background: #e0e0e0;
  cursor: pointer;
  transition: all 0.3s;
}

.tab-btn.active {
  background: #007bff;
  color: white;
}

.nodes-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.node-card {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.node-card h3 {
  margin: 0 0 15px 0;
  color: #333;
  border-bottom: 2px solid #007bff;
  padding-bottom: 5px;
}

.sensor-data {
  margin-bottom: 20px;
}

.sensor-data p {
  margin: 5px 0;
  display: flex;
  justify-content: space-between;
}

.value {
  font-weight: bold;
  color: #007bff;
}

.control-section {
  margin-bottom: 15px;
}

.status {
  font-weight: bold;
  padding: 2px 8px;
  border-radius: 4px;
}

.status.on {
  background: #d4edda;
  color: #155724;
}

.status.off {
  background: #f8d7da;
  color: #721c24;
}

.control-buttons {
  display: flex;
  gap: 8px;
  margin-top: 10px;
  flex-wrap: wrap;
}

.btn {
  padding: 8px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.3s;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-on {
  background: #28a745;
  color: white;
}

.btn-off {
  background: #dc3545;
  color: white;
}

.btn-auto {
  background: #6c757d;
  color: white;
}

.btn-chart {
  width: 100%;
  background: #17a2b8;
  color: white;
  margin-top: 10px;
}

.btn-chart.active {
  background: #007bff;
}

.chart-section {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.chart-container h3 {
  margin-top: 0;
  color: #333;
}

@media (max-width: 768px) {
  .nodes-grid {
    grid-template-columns: 1fr;
  }
  
  .control-buttons {
    flex-direction: column;
  }
}
```

## 5. Aplikasi Mobile (React Native)

### 5.1 Setup React Native Project

```bash
# Install React Native CLI
npm install -g @react-native-community/cli

# Create new project
npx react-native init HydroponicsApp
cd HydroponicsApp

# Install dependencies
npm install axios react-native-vector-icons react-native-chart-kit react-native-svg
```

### 5.2 Main App Component

```jsx
// App.js
import React, { useState, useEffect } from 'react';
import {
  SafeAreaView,
  ScrollView,
  StyleSheet,
  Text,
  View,
  TouchableOpacity,
  RefreshControl,
  Alert,
  Dimensions,
} from 'react-native';
import axios from 'axios';
import { LineChart } from 'react-native-chart-kit';

const API_BASE_URL = 'http://192.168.1.100:5000'; // IP Raspberry Pi
const screenWidth = Dimensions.get('window').width;

const App = () => {
  const [exhaustData, setExhaustData] = useState([]);
  const [nutrientData, setNutrientData] = useState([]);
  const [activeTab, setActiveTab] = useState('exhaust');
  const [refreshing, setRefreshing] = useState(false);
  const [selectedNode, setSelectedNode] = useState(1);
  const [chartData, setChartData] = useState(null);

  useEffect(() => {
    fetchData();
    const interval = setInterval(fetchData, 15000); // Update setiap 15 detik
    return () => clearInterval(interval);
  }, []);

  useEffect(() => {
    fetchChartData(selectedNode);
  }, [selectedNode, activeTab]);

  const fetchData = async () => {
    try {
      const [exhaustResponse, nutrientResponse] = await Promise.all([
        axios.get(`${API_BASE_URL}/api/exhaust/latest`),
        axios.get(`${API_BASE_URL}/api/nutrient/latest`)
      ]);
      
      setExhaustData(exhaustResponse.data);
      setNutrientData(nutrientResponse.data);
    } catch (error) {
      console.error('Error fetching data:', error);
      Alert.alert('Error', 'Failed to fetch data from server');
    }
  };

  const fetchChartData = async (nodeId) => {
    try {
      const endpoint = activeTab === 'exhaust' 
        ? `/api/exhaust/history/${nodeId}?hours=12`
        : `/api/nutrient/history/${nodeId}?hours=12`;
      
      const response = await axios.get(`${API_BASE_URL}${endpoint}`);
      const data = response.data.reverse().slice(-20); // Last 20 points
      
      if (data.length > 0) {
        const labels = data.map((_, index) => index.toString());
        const datasets = [];
        
        if (activeTab === 'exhaust') {
          datasets.push({
            data: data.map(item => item.temperature || 0),
            color: (opacity = 1) => `rgba(255, 99, 132, ${opacity})`,
            strokeWidth: 2,
          });
        } else {
          datasets.push({
            data: data.map(item => item.tdsValue || 0),
            color: (opacity = 1) => `rgba(54, 162, 235, ${opacity})`,
            strokeWidth: 2,
          });
        }
        
        setChartData({ labels, datasets });
      }
    } catch (error) {
      console.error('Error fetching chart data:', error);
    }
  };

  const onRefresh = async () => {
    setRefreshing(true);
    await fetchData();
    await fetchChartData(selectedNode);
    setRefreshing(false);
  };

  const toggleRelay = async (nodeId, newState) => {
    try {
      const endpoint = activeTab === 'exhaust' 
        ? `/api/exhaust/control/${nodeId}`
        : `/api/nutrient/control/${nodeId}`;
      
      await axios.post(`${API_BASE_URL}${endpoint}`, {
        manualControl: true,
        relayState: newState
      });
      
      Alert.alert('Success', `Node ${nodeId} ${newState ? 'turned ON' : 'turned OFF'}`);
      fetchData();
    } catch (error) {
      Alert.alert('Error', 'Failed to control relay');
    }
  };

  const resetToAuto = async (nodeId) => {
    try {
      const endpoint = activeTab === 'exhaust' 
        ? `/api/exhaust/control/${nodeId}`
        : `/api/nutrient/control/${nodeId}`;
      
      await axios.post(`${API_BASE_URL}${endpoint}`, {
        manualControl: false
      });
      
      Alert.alert('Success', `Node ${nodeId} set to AUTO mode`);
      fetchData();
    } catch (error) {
      Alert.alert('Error', 'Failed to set auto mode');
    }
  };

  const renderNodeCard = (node) => (
    <View key={node.nodeId} style={styles.nodeCard}>
      <Text style={styles.nodeTitle}>Node {node.nodeId}</Text>
      
      {activeTab === 'exhaust' ? (
        <View style={styles.sensorData}>
          <Text style={styles.sensorText}>
            Temperature: <Text style={styles.valueText}>{node.temperature?.toFixed(1)}°C</Text>
          </Text>
          <Text style={styles.sensorText}>
            Humidity: <Text style={styles.valueText}>{node.humidity?.toFixed(1)}%</Text>
          </Text>
        </View>
      ) : (
        <View style={styles.sensorData}>
          <Text style={styles.sensorText}>
            TDS: <Text style={styles.valueText}>{node.tdsValue?.toFixed(0)} ppm</Text>
          </Text>
        </View>
      )}
      
      <View style={styles.statusContainer}>
        <Text style={styles.statusLabel}>Status: </Text>
        <Text style={[styles.statusText, node.relayState ? styles.statusOn : styles.statusOff]}>
          {node.relayState ? 'ON' : 'OFF'}
        </Text>
      </View>
      
      <View style={styles.buttonContainer}>
        <TouchableOpacity 
          style={[styles.button, styles.onButton, node.relayState && styles.disabledButton]}
          onPress={() => toggleRelay(node.nodeId, true)}
          disabled={node.relayState}
        >
          <Text style={styles.buttonText}>ON</Text>
        </TouchableOpacity>
        
        <TouchableOpacity 
          style={[styles.button, styles.offButton, !node.relayState && styles.disabledButton]}
          onPress={() => toggleRelay(node.nodeId, false)}
          disabled={!node.relayState}
        >
          <Text style={styles.buttonText}>OFF</Text>
        </TouchableOpacity>
        
        <TouchableOpacity 
          style={[styles.button, styles.autoButton]}
          onPress={() => resetToAuto(node.nodeId)}
        >
          <Text style={styles.buttonText}>AUTO</Text>
        </TouchableOpacity>
      </View>
      
      <TouchableOpacity 
        style={[styles.chartButton, selectedNode === node.nodeId && styles.activeChartButton]}
        onPress={() => setSelectedNode(node.nodeId)}
      >
        <Text style={styles.chartButtonText}>View Chart</Text>
      </TouchableOpacity>
    </View>
  );

  const currentData = activeTab === 'exhaust' ? exhaustData : nutrientData;

  return (
    <SafeAreaView style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>Hydroponics IoT</Text>
        <View style={styles.tabContainer}>
          <TouchableOpacity 
            style={[styles.tab, activeTab === 'exhaust' && styles.activeTab]}
            onPress={() => setActiveTab('exhaust')}
          >
            <Text style={[styles.tabText, activeTab === 'exhaust' && styles.activeTabText]}>
              Exhaust
            </Text>
          </TouchableOpacity>
          <TouchableOpacity 
            style={[styles.tab, activeTab === 'nutrient' && styles.activeTab]}
            onPress={() => setActiveTab('nutrient')}
          >
            <Text style={[styles.tabText, activeTab === 'nutrient' && styles.activeTabText]}>
              Nutrient
            </Text>
          </TouchableOpacity>
        </View>
      </View>

      <ScrollView 
        style={styles.scrollView}
        refreshControl={<RefreshControl refreshing={refreshing} onRefresh={onRefresh} />}
      >
        {currentData.map(renderNodeCard)}
        
        {chartData && (
          <View style={styles.chartContainer}>
            <Text style={styles.chartTitle}>
              Node {selectedNode} - {activeTab === 'exhaust' ? 'Temperature' : 'TDS'} History
            </Text>
            <LineChart
              data={chartData}
              width={screenWidth - 40}
              height={220}
              chartConfig={{
                backgroundColor: '#ffffff',
                backgroundGradientFrom: '#ffffff',
                backgroundGradientTo: '#ffffff',
                decimalPlaces: 1,
                color: (opacity = 1) => `rgba(0, 123, 255, ${opacity})`,
                labelColor: (opacity = 1) => `rgba(0, 0, 0, ${opacity})`,
                style: {
                  borderRadius: 16,
                },
                propsForDots: {
                  r: '3',
                  strokeWidth: '1',
                  stroke: '#007bff',
                },
              }}
              bezier
              style={styles.chart}
            />
          </View>
        )}
      </ScrollView>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    backgroundColor: '#ffffff',
    padding: 20,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 15,
  },
  tabContainer: {
    flexDirection: 'row',
    backgroundColor: '#f0f0f0',
    borderRadius: 8,
    padding: 4,
  },
  tab: {
    flex: 1,
    paddingVertical: 10,
    alignItems: 'center',
    borderRadius: 6,
  },
  activeTab: {
    backgroundColor: '#007bff',
  },
  tabText: {
    fontSize: 16,
    color: '#666',
  },
  activeTabText: {
    color: '#ffffff',
    fontWeight: 'bold',
  },
  scrollView: {
    flex: 1,
    padding: 20,
  },
  nodeCard: {
    backgroundColor: '#ffffff',
    padding: 20,
    marginBottom: 20,
    borderRadius: 12,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  nodeTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 15,
    borderBottomWidth: 2,
    borderBottomColor: '#007bff',
    paddingBottom: 5,
  },
  sensorData: {
    marginBottom: 15,
  },
  sensorText: {
    fontSize: 16,
    color: '#666',
    marginBottom: 5,
  },
  valueText: {
    fontWeight: 'bold',
    color: '#007bff',
  },
  statusContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 15,
  },
  statusLabel: {
    fontSize: 16,
    color: '#666',
  },
  statusText: {
    fontSize: 16,
    fontWeight: 'bold',
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: 4,
  },
  statusOn: {
    backgroundColor: '#d4edda',
    color: '#155724',
  },
  statusOff: {
    backgroundColor: '#f8d7da',
    color: '#721c24',
  },
  buttonContainer: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 15,
  },
  button: {
    flex: 1,
    paddingVertical: 10,
    marginHorizontal: 4,
    borderRadius: 6,
    alignItems: 'center',
  },
  onButton: {
    backgroundColor: '#28a745',
  },
  offButton: {
    backgroundColor: '#dc3545',
  },
  autoButton: {
    backgroundColor: '#6c757d',
  },
  disabledButton: {
    opacity: 0.5,
  },
  buttonText: {
    color: '#ffffff',
    fontWeight: 'bold',
    fontSize: 14,
  },
  chartButton: {
    backgroundColor: '#17a2b8',
    paddingVertical: 12,
    borderRadius: 6,
    alignItems: 'center',
  },
  activeChartButton: {
    backgroundColor: '#007bff',
  },
  chartButtonText: {
    color: '#ffffff',
    fontWeight: 'bold',
    fontSize: 16,
  },
  chartContainer: {
    backgroundColor: '#ffffff',
    padding: 20,
    borderRadius: 12,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    marginBottom: 20,
  },
  chartTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 15,
    textAlign: 'center',
  },
  chart: {
    borderRadius: 16,
  },
});

export default App;
```

## 6. Langkah Instalasi Lengkap

### 6.1 Setup ESP32

```bash
# Install Arduino IDE
# Download dari: https://www.arduino.cc/en/software

# Install ESP32 Board Package:
# 1. File → Preferences
# 2. Additional Board Manager URLs: 
#    https://dl.espressif.com/dl/package_esp32_index.json
# 3. Tools → Board → Boards Manager
# 4. Search "ESP32" dan install

# Install Required Libraries:
# 1. DHT sensor library by Adafruit
# 2. ArduinoJson by Benoit Blanchon
# 3. WiFi library (built-in)
```

### 6.2 Konfigurasi WiFi untuk Semua ESP32

```cpp
// wifi_config.h (buat file terpisah untuk konfigurasi WiFi)
#ifndef WIFI_CONFIG_H
#define WIFI_CONFIG_H

// WiFi Configuration
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Server Configuration
const char* serverURL = "http://192.168.1.100:5000"; // IP Raspberry Pi

// Node Configuration (ubah untuk setiap ESP32)
// Exhaust nodes: 1-4
// Nutrient nodes: 1-4
int nodeId = 1; // UBAH INI UNTUK SETIAP NODE

#endif
```

### 6.3 Setup Raspberry Pi Lengkap

```bash
#!/bin/bash
# setup_raspberry_pi.sh

echo "=== Hydroponics IoT - Raspberry Pi Setup ==="

# Update sistem
echo "Updating system..."
sudo apt update && sudo apt upgrade -y

# Install dependencies
echo "Installing dependencies..."
sudo apt install -y python3 python3-pip python3-venv postgresql postgresql-contrib git

# Create project directory
echo "Creating project directory..."
cd /home/pi
mkdir hydroponics-iot
cd hydroponics-iot

# Create Python virtual environment
echo "Setting up Python environment..."
python3 -m venv venv
source venv/bin/activate

# Install Python packages
pip install flask flask-cors psycopg2-binary python-dotenv

# Setup PostgreSQL
echo "Setting up PostgreSQL..."
sudo -u postgres createdb hydroponics_iot
sudo -u postgres psql -c "CREATE USER iot_user WITH ENCRYPTED PASSWORD 'HydroPass2024';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE hydroponics_iot TO iot_user;"

# Create app.py (backend server)
cat > app.py << 'EOF'
# Kode Flask yang sudah dibuat sebelumnya
# (Copy dari section 3.3)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==2.3.3
Flask-CORS==4.0.0
psycopg2-binary==2.9.9
python-dotenv==1.0.0
EOF

# Create .env file
cat > .env << 'EOF'
DB_HOST=localhost
DB_NAME=hydroponics_iot
DB_USER=iot_user
DB_PASSWORD=HydroPass2024
FLASK_ENV=production
EOF

# Create systemd service
sudo tee /etc/systemd/system/hydroponics-api.service > /dev/null << 'EOF'
[Unit]
Description=Hydroponics IoT API Server
After=network.target postgresql.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/hydroponics-iot
Environment=PATH=/home/pi/hydroponics-iot/venv/bin
ExecStart=/home/pi/hydroponics-iot/venv/bin/python app.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable hydroponics-api.service
sudo systemctl start hydroponics-api.service

echo "=== Setup Complete ==="
echo "API Server running on: http://$(hostname -I | awk '{print $1}'):5000"
echo "Check service status: sudo systemctl status hydroponics-api.service"
```

### 6.4 Setup Web Application

```bash
#!/bin/bash
# setup_web_app.sh

echo "=== Setting up Web Application ==="

# Install Node.js dan npm
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Create web application
npx create-react-app hydroponics-web
cd hydroponics-web

# Install dependencies
npm install axios recharts

# Replace default App.js with our code
# (Copy kode React yang sudah dibuat)

# Build for production
npm run build

# Serve using simple HTTP server
npm install -g serve
serve -s build -l 3000

echo "Web application running on: http://localhost:3000"
```

### 6.5 Setup Mobile Application

```bash
#!/bin/bash
# setup_mobile_app.sh

echo "=== Setting up Mobile Application ==="

# Install React Native CLI
npm install -g @react-native-community/cli

# Create mobile app
npx react-native init HydroponicsApp
cd HydroponicsApp

# Install dependencies
npm install axios react-native-vector-icons react-native-chart-kit react-native-svg

# Setup vector icons (Android)
npx react-native link react-native-vector-icons

# For iOS (if needed)
cd ios && pod install && cd ..

echo "Mobile app created successfully"
echo "To run: npx react-native run-android (or run-ios)"
```

## 7. Keamanan dan Protokol Komunikasi

### 7.1 Konfigurasi Keamanan WiFi

```cpp
// security_config.h
#ifndef SECURITY_CONFIG_H
#define SECURITY_CONFIG_H

// API Key untuk autentikasi sederhana
const char* apiKey = "HydroIoT2024SecureKey";

// Timeout settings
const int httpTimeout = 10000; // 10 seconds
const int maxRetries = 3;

// WiFi reconnection settings
const int wifiReconnectDelay = 5000; // 5 seconds

bool authenticateRequest(String key) {
    return key.equals(apiKey);
}

#endif
```

### 7.2 Enhanced ESP32 Code dengan Security

```cpp
// Tambahan untuk ESP32 code (exhaust/nutrient)
void sendSecureData(float value1, float value2, bool relayState) {
    if (WiFi.status() == WL_CONNECTED) {
        HTTPClient http;
        http.setTimeout(httpTimeout);
        
        String endpoint = (nodeType == "exhaust") ? "/api/exhaust/data" : "/api/nutrient/data";
        http.begin(String(serverURL) + endpoint);
        http.addHeader("Content-Type", "application/json");
        http.addHeader("Authorization", apiKey);
        
        StaticJsonDocument<300> doc;
        doc["nodeId"] = nodeId;
        doc["apiKey"] = apiKey;
        doc["timestamp"] = millis();
        
        if (nodeType == "exhaust") {
            doc["temperature"] = value1;
            doc["humidity"] = value2;
        } else {
            doc["tdsValue"] = value1;
        }
        doc["relayState"] = relayState;
        
        String jsonString;
        serializeJson(doc, jsonString);
        
        int attempt = 0;
        while (attempt < maxRetries) {
            int httpResponseCode = http.POST(jsonString);
            
            if (httpResponseCode > 0) {
                Serial.println("Data sent successfully");
                break;
            } else {
                attempt++;
                Serial.printf("HTTP Error: %d, Retry %d/%d\n", httpResponseCode, attempt, maxRetries);
                delay(2000);
            }
        }
        
        http.end();
    } else {
        // WiFi reconnection logic
        WiFi.reconnect();
        delay(wifiReconnectDelay);
    }
}
```

### 7.3 Enhanced Backend Security

```python
# Enhanced Flask app with security
from functools import wraps
import hashlib
import secrets

# Security configuration
API_KEY = "HydroIoT2024SecureKey"
SESSION_SECRET = secrets.token_hex(32)

def require_api_key(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        api_key = request.headers.get('Authorization') or request.json.get('apiKey')
        if api_key != API_KEY:
            return jsonify({'error': 'Invalid API key'}), 401
        return f(*args, **kwargs)
    return decorated_function

# Apply to all ESP32 endpoints
@app.route('/api/exhaust/data', methods=['POST'])
@require_api_key
def receive_exhaust_data():
    # Existing code...
    pass

@app.route('/api/nutrient/data', methods=['POST'])
@require_api_key
def receive_nutrient_data():
    # Existing code...
    pass

# Rate limiting untuk web/mobile access
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["1000 per hour"]
)

@app.route('/api/exhaust/latest', methods=['GET'])
@limiter.limit("60 per minute")
def get_latest_exhaust_data():
    # Existing code...
    pass
```

## 8. Pengujian Sistem dan Troubleshooting

### 8.1 Script Testing Otomatis

```python
# test_system.py
import requests
import time
import json

RASPBERRY_PI_IP = "192.168.1.100"
BASE_URL = f"http://{RASPBERRY_PI_IP}:5000"

def test_api_endpoints():
    """Test semua API endpoints"""
    
    print("=== Testing API Endpoints ===")
    
    # Test endpoints
    endpoints = [
        "/api/exhaust/latest",
        "/api/nutrient/latest",
        "/api/exhaust/history/1",
        "/api/nutrient/history/1"
    ]
    
    for endpoint in endpoints:
        try:
            response = requests.get(f"{BASE_URL}{endpoint}", timeout=5)
            print(f"✓ {endpoint}: {response.status_code}")
        except Exception as e:
            print(f"✗ {endpoint}: {str(e)}")

def test_control_endpoints():
    """Test control endpoints"""
    
    print("\n=== Testing Control Endpoints ===")
    
    # Test manual control
    control_data = {
        "manualControl": True,
        "relayState": True
    }
    
    try:
        response = requests.post(f"{BASE_URL}/api/exhaust/control/1", 
                               json=control_data, timeout=5)
        print(f"✓ Manual control: {response.status_code}")
    except Exception as e:
        print(f"✗ Manual control: {str(e)}")

def simulate_esp32_data():
    """Simulate ESP32 sending data"""
    
    print("\n=== Simulating ESP32 Data ===")
    
    # Simulate exhaust data
    exhaust_data = {
        "nodeId": 99,  # Test node
        "temperature": 25.5,
        "humidity": 60.0,
        "relayState": False,
        "timestamp": int(time.time() * 1000)
    }
    
    try:
        response = requests.post(f"{BASE_URL}/api/exhaust/data", 
                               json=exhaust_data, 
                               headers={"Authorization": "HydroIoT2024SecureKey"},
                               timeout=5)
        print(f"✓ Exhaust data simulation: {response.status_code}")
    except Exception as e:
        print(f"✗ Exhaust data simulation: {str(e)}")

if __name__ == "__main__":
    test_api_endpoints()
    test_control_endpoints()
    simulate_esp32_data()
```

### 8.2 Monitoring Script

```bash
#!/bin/bash
# monitor_system.sh

echo "=== Hydroponics IoT System Monitor ==="

# Check Raspberry Pi services
echo "Checking services..."
sudo systemctl status hydroponics-api.service --no-pager -l

# Check database
echo -e "\nChecking database..."
sudo -u postgres psql -d hydroponics_iot -c "SELECT COUNT(*) FROM exhaust_data;"
sudo -u postgres psql -d hydroponics_iot -c "SELECT COUNT(*) FROM nutrient_data;"

# Check network connectivity
echo -e "\nChecking network..."
ping -c 3 8.8.8.8

# Check API endpoints
echo -e "\nTesting API..."
curl -s http://localhost:5000/api/exhaust/latest | python3 -m json.tool

# Check system resources
echo -e "\nSystem resources:"
free -h
df -h /

echo -e "\n=== Monitor Complete ==="
```

### 8.3 Common Issues dan Solutions

```markdown
### Troubleshooting Guide

#### 1. ESP32 tidak dapat connect ke WiFi
**Symptoms**: ESP32 stuck di "Connecting to WiFi..."
**Solutions**:
- Periksa SSID dan password WiFi
- Pastikan WiFi 2.4GHz (ESP32 tidak support 5GHz)
- Reset ESP32 dan flash ulang
- Periksa jarak dari router

#### 2. Data tidak sampai ke Raspberry Pi
**Symptoms**: Database kosong, tidak ada data masuk
**Solutions**:
- Periksa IP address Raspberry Pi di ESP32 code
- Test dengan curl: `curl -X POST http://RPI_IP:5000/api/exhaust/data`
- Periksa firewall Raspberry Pi
- Check service status: `sudo systemctl status hydroponics-api.service`

#### 3. Web/Mobile app tidak dapat akses data
**Symptoms**: "Network Error" atau "Failed to fetch"
**Solutions**:
