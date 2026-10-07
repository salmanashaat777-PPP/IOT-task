// ============================================================
// ESP32 IOT GATEWAY - FINAL
//
// Receives:
//   Temperature
//   Humidity
//   LDR
//
// Via:
//   ESP-NOW CHANNEL 11
//
// Controls:
//   LED
//   Relay -> DC Motor
//
// POWER MANAGEMENT:
//   Wi-Fi modem sleep enabled
//   CPU reduced to 80 MHz
//
// IMPORTANT:
//   Gateway DOES NOT deep sleep.
//   It must remain awake to receive ESP-NOW.
// ============================================================

#include <Arduino.h>
#include <WiFi.h>
#include <esp_now.h>

// ============================================================
// WIFI
// ============================================================

#define WIFI_SSID       "YOUR_WIFI_NAME"
#define WIFI_PASSWORD   "YOUR_WIFI_PASSWORD"

// Gateway and sensor MUST use same channel
#define EXPECTED_CHANNEL 11

// ============================================================
// OUTPUT PINS
// ============================================================

#define LED_PIN     26
#define RELAY_PIN   27

// ============================================================
// RELAY LOGIC
//
// Most relay modules are ACTIVE LOW.
//
// true  = LOW turns relay ON
// false = HIGH turns relay ON
// ============================================================

#define RELAY_ACTIVE_LOW true

// ============================================================
// TEMPERATURE CONTROL
// ============================================================

// Motor ON at or above 27 C
#define TEMP_ON_THRESHOLD   27.0

// Motor OFF at or below 25.5 C
#define TEMP_OFF_THRESHOLD  25.5

// Between these values, keep previous motor state.

// ============================================================
// LDR CONTROL
//
// Your actual LDR behavior:
//
// DARKNESS -> HIGH ADC VALUE
// LIGHT    -> LOW ADC VALUE
//
// Therefore:
//
// LDR > 2500  -> DARK -> LED ON
// LDR <= 2500 -> LIGHT -> LED OFF
// ============================================================

#define DARK_THRESHOLD 2500

// ============================================================
// DATA PACKET
//
// MUST MATCH SENSOR EXACTLY
// ============================================================

typedef struct __attribute__((packed)) {

    uint8_t nodeID;

    float temperature;
    float humidity;

    uint16_t lightLevel;

    uint32_t bootCount;

    int8_t rssi;

    uint8_t batteryPct;

    bool sensorValid;

} SensorPacket_t;

// ============================================================
// GLOBAL VARIABLES
// ============================================================

SensorPacket_t receivedPacket;

volatile bool newDataAvailable = false;

bool motorState = false;

bool ledState = false;

unsigned long lastPacketTime = 0;

// ============================================================
// RELAY CONTROL
// ============================================================

void setMotor(bool on) {

    motorState = on;

    if (RELAY_ACTIVE_LOW) {

        digitalWrite(
            RELAY_PIN,
            on ? LOW : HIGH
        );

    } else {

        digitalWrite(
            RELAY_PIN,
            on ? HIGH : LOW
        );
    }

    Serial.print("[MOTOR] ");

    if (on) {
        Serial.println("ON");
    } else {
        Serial.println("OFF");
    }
}

// ============================================================
// LED CONTROL
// ============================================================

void setLED(bool on) {

    ledState = on;

    digitalWrite(
        LED_PIN,
        on ? HIGH : LOW
    );

    Serial.print("[LED] ");

    if (on) {
        Serial.println("ON");
    } else {
        Serial.println("OFF");
    }
}

// ============================================================
// ESP-NOW RECEIVE CALLBACK
//
// ESP32 Arduino Core 3.x
// ============================================================

void onDataReceived(
    const esp_now_recv_info_t *info,
    const uint8_t *incomingData,
    int len
) {

    // --------------------------------------------------------
    // VERIFY PACKET SIZE
    // --------------------------------------------------------

    if (len != sizeof(SensorPacket_t)) {

        Serial.println();
        Serial.println("[ESP-NOW ERROR]");

        Serial.print("Received size: ");
        Serial.println(len);

        Serial.print("Expected size: ");
        Serial.println(
            sizeof(SensorPacket_t)
        );

        return;
    }

    // --------------------------------------------------------
    // COPY DATA
    // --------------------------------------------------------

    memcpy(
        &receivedPacket,
        incomingData,
        sizeof(receivedPacket)
    );

    lastPacketTime = millis();

    newDataAvailable = true;
}

// ============================================================
// CONNECT TO WIFI
// ============================================================

bool connectWiFi() {

    Serial.println();
    Serial.println("========================================");
    Serial.println("           WIFI CONNECTION");
    Serial.println("========================================");

    WiFi.mode(WIFI_STA);

    WiFi.begin(
        WIFI_SSID,
        WIFI_PASSWORD
    );

    Serial.print("[WiFi] Connecting to: ");
    Serial.println(WIFI_SSID);

    Serial.print("[WiFi] Connecting");

    unsigned long startTime = millis();

    while (
        WiFi.status() != WL_CONNECTED &&
        millis() - startTime < 20000
    ) {

        Serial.print(".");
        delay(500);
    }

    Serial.println();

    // --------------------------------------------------------
    // FAILED
    // --------------------------------------------------------

    if (WiFi.status() != WL_CONNECTED) {

        Serial.println();
        Serial.println(
            "[ERROR] Wi-Fi connection failed."
        );

        return false;
    }

    // --------------------------------------------------------
    // CONNECTED
    // --------------------------------------------------------

    Serial.println();
    Serial.println("[WiFi] CONNECTED");

    Serial.print("[WiFi] IP Address : ");
    Serial.println(WiFi.localIP());

    Serial.print("[WiFi] Gateway    : ");
    Serial.println(WiFi.gatewayIP());

    Serial.print("[WiFi] Subnet     : ");
    Serial.println(WiFi.subnetMask());

    Serial.print("[WiFi] Channel    : ");
    Serial.println(WiFi.channel());

    Serial.print("[WiFi] RSSI       : ");
    Serial.print(WiFi.RSSI());
    Serial.println(" dBm");

    Serial.print("[Gateway] MAC Address: ");
    Serial.println(WiFi.macAddress());

    return true;
}

// ============================================================
// POWER MANAGEMENT
// ============================================================

void initPowerManagement() {

    Serial.println();
    Serial.println("========================================");
    Serial.println("          POWER MANAGEMENT");
    Serial.println("========================================");

    // --------------------------------------------------------
    // Wi-Fi MODEM POWER SAVE
    // --------------------------------------------------------

    WiFi.setSleep(true);

    Serial.println(
        "[POWER] Wi-Fi power saving enabled."
    );

    // --------------------------------------------------------
    // REDUCE CPU FREQUENCY
    // --------------------------------------------------------

    setCpuFrequencyMhz(80);

    Serial.print(
        "[POWER] CPU frequency: "
    );

    Serial.print(
        getCpuFrequencyMhz()
    );

    Serial.println(" MHz");

    Serial.println(
        "[POWER] Gateway remains awake for ESP-NOW."
    );
}

// ============================================================
// INITIALIZE ESP-NOW
// ============================================================

bool initESPNow() {

    Serial.println();
    Serial.println("========================================");
    Serial.println("           ESP-NOW SETUP");
    Serial.println("========================================");

    int wifiChannel =
        WiFi.channel();

    Serial.print(
        "[ESP-NOW] Wi-Fi channel = "
    );

    Serial.println(
        wifiChannel
    );

    Serial.print(
        "[ESP-NOW] Sensor channel = "
    );

    Serial.println(
        EXPECTED_CHANNEL
    );

    // --------------------------------------------------------
    // CHANNEL CHECK
    // --------------------------------------------------------

    if (
        wifiChannel != EXPECTED_CHANNEL
    ) {

        Serial.println();
        Serial.println("========================================");
        Serial.println("              WARNING");
        Serial.println("========================================");

        Serial.println(
            "Wi-Fi channel does NOT match sensor."
        );

        Serial.println();

        Serial.print(
            "Gateway Wi-Fi channel : "
        );

        Serial.println(
            wifiChannel
        );

        Serial.print(
            "Sensor channel        : "
        );

        Serial.println(
            EXPECTED_CHANNEL
        );

        return false;
    }

    // --------------------------------------------------------
    // ESP-NOW INITIALIZATION
    // --------------------------------------------------------

    esp_err_t result =
        esp_now_init();

    if (result != ESP_OK) {

        Serial.print(
            "[ERROR] ESP-NOW initialization failed. Code: "
        );

        Serial.println(
            result
        );

        return false;
    }

    // --------------------------------------------------------
    // RECEIVE CALLBACK
    // --------------------------------------------------------

    esp_now_register_recv_cb(
        onDataReceived
    );

    Serial.println(
        "[ESP-NOW] Initialized successfully."
    );

    Serial.println(
        "[ESP-NOW] Waiting for sensor packets..."
    );

    return true;
}

// ============================================================
// PROCESS SENSOR DATA
// ============================================================

void processSensorData() {

    newDataAvailable = false;

    Serial.println();
    Serial.println("========================================");
    Serial.println("       ESP-NOW PACKET RECEIVED");
    Serial.println("========================================");

    // --------------------------------------------------------
    // SENSOR DATA
    // --------------------------------------------------------

    Serial.print("Node ID      : ");
    Serial.println(
        receivedPacket.nodeID
    );

    Serial.print("Temperature  : ");
    Serial.print(
        receivedPacket.temperature
    );

    Serial.println(" C");

    Serial.print("Humidity     : ");
    Serial.print(
        receivedPacket.humidity
    );

    Serial.println(" %");

    Serial.print("Light Level  : ");
    Serial.println(
        receivedPacket.lightLevel
    );

    Serial.print("Packet Count : ");
    Serial.println(
        receivedPacket.bootCount
    );

    Serial.print("Battery      : ");
    Serial.print(
        receivedPacket.batteryPct
    );

    Serial.println(" %");

    Serial.print("Sensor Valid : ");

    if (
        receivedPacket.sensorValid
    ) {

        Serial.println("YES");

    } else {

        Serial.println("NO");
    }

    // ========================================================
    // INVALID DHT DATA
    // ========================================================

    if (
        !receivedPacket.sensorValid
    ) {

        Serial.println();
        Serial.println(
            "[WARNING] Invalid DHT reading."
        );

        // Safety
        setMotor(false);

        return;
    }

    // ========================================================
    // MOTOR CONTROL
    // ========================================================

    Serial.println();
    Serial.println("----------------------------------------");
    Serial.println("             CONTROL");
    Serial.println("----------------------------------------");

    // --------------------------------------------------------
    // MOTOR ON
    // --------------------------------------------------------

    if (
        receivedPacket.temperature
        >= TEMP_ON_THRESHOLD
    ) {

        if (!motorState) {

            Serial.println(
                "[CONTROL] Temperature HIGH"
            );

            setMotor(true);
        }
    }

    // --------------------------------------------------------
    // MOTOR OFF
    // --------------------------------------------------------

    else if (
        receivedPacket.temperature
        <= TEMP_OFF_THRESHOLD
    ) {

        if (motorState) {

            Serial.println(
                "[CONTROL] Temperature NORMAL"
            );

            setMotor(false);
        }
    }

    // ========================================================
    // LED CONTROL
    //
    // HIGH LDR = DARK
    // LOW LDR  = BRIGHT
    // ========================================================

    if (
        receivedPacket.lightLevel
        > DARK_THRESHOLD
    ) {

        if (!ledState) {

            Serial.println(
                "[CONTROL] Environment DARK"
            );

            setLED(true);
        }

    } else {

        if (ledState) {

            Serial.println(
                "[CONTROL] Environment BRIGHT"
            );

            setLED(false);
        }
    }

    // --------------------------------------------------------
    // STATES
    // --------------------------------------------------------

    Serial.println();

    Serial.print("Motor State : ");

    Serial.println(
        motorState ? "ON" : "OFF"
    );

    Serial.print("LED State   : ");

    Serial.println(
        ledState ? "ON" : "OFF"
    );

    Serial.println(
        "========================================"
    );
}

// ============================================================
// SETUP
// ============================================================

void setup() {

    Serial.begin(115200);

    delay(1000);

    Serial.println();
    Serial.println("========================================");
    Serial.println("          ESP32 IOT GATEWAY");
    Serial.println("========================================");

    // ========================================================
    // GPIO
    // ========================================================

    Serial.println(
        "[GPIO] Initializing..."
    );

    pinMode(
        LED_PIN,
        OUTPUT
    );

    pinMode(
        RELAY_PIN,
        OUTPUT
    );

    // --------------------------------------------------------
    // LED OFF
    // --------------------------------------------------------

    digitalWrite(
        LED_PIN,
        LOW
    );

    ledState = false;

    Serial.println(
        "[GPIO] LED initialized."
    );

    // --------------------------------------------------------
    // MOTOR OFF
    // --------------------------------------------------------

    setMotor(false);

    Serial.println(
        "[GPIO] Motor initialized."
    );

    // ========================================================
    // WIFI
    // ========================================================

    if (!connectWiFi()) {

        Serial.println();
        Serial.println(
            "Gateway stopped because Wi-Fi failed."
        );

        while (true) {
            delay(1000);
        }
    }

    // ========================================================
    // POWER MANAGEMENT
    // ========================================================

    initPowerManagement();

    // ========================================================
    // ESP-NOW
    // ========================================================

    if (!initESPNow()) {

        Serial.println();
        Serial.println(
            "Gateway stopped because ESP-NOW failed."
        );

        while (true) {
            delay(1000);
        }
    }

    // ========================================================
    // READY
    // ========================================================

    Serial.println();
    Serial.println("========================================");
    Serial.println("          GATEWAY READY");
    Serial.println("========================================");

    Serial.println(
        "Waiting for sensor ESP32..."
    );
}

// ============================================================
// LOOP
// ============================================================

void loop() {

    // Process packets outside callback
    if (newDataAvailable) {

        processSensorData();
    }

    // Small idle period
    delay(50);
}
