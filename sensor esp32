// ============================================================
// ESP32 SENSOR NODE - FINAL
//
// DHT11 + LDR
//       |
//       v
//   ESP32 SENSOR
//       |
//       | ESP-NOW - CHANNEL 11
//       v
//   ESP32 GATEWAY
//
// POWER MANAGEMENT:
// Wake -> Read -> Send -> Deep Sleep 10 sec -> Repeat
// ============================================================

#include <Arduino.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <esp_sleep.h>
#include <DHT.h>

// ============================================================
// SENSOR PINS
// ============================================================

#define DHT_PIN     4
#define DHT_TYPE    DHT11
#define LDR_PIN     34

DHT dht(DHT_PIN, DHT_TYPE);

// ============================================================
// POWER MANAGEMENT
// ============================================================

#define SLEEP_SECONDS 10

// Survives deep sleep
RTC_DATA_ATTR uint32_t bootCount = 0;

// ============================================================
// ESP-NOW SETTINGS
// ============================================================

// Gateway MAC
uint8_t GATEWAY_MAC[] = {
    0x84, 0x1F, 0xE8, 0x16, 0xE5, 0x00
};

// MUST MATCH GATEWAY WI-FI CHANNEL
#define ESP_NOW_CHANNEL 11

// ============================================================
// DATA PACKET
//
// THIS MUST MATCH THE GATEWAY EXACTLY
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
// SEND STATUS
// ============================================================

volatile bool sendComplete = false;
volatile bool sendSuccess = false;

// ============================================================
// ESP-NOW SEND CALLBACK
// ============================================================

void onDataSent(
    const wifi_tx_info_t *info,
    esp_now_send_status_t status
) {
    sendSuccess =
        (status == ESP_NOW_SEND_SUCCESS);

    sendComplete = true;
}

// ============================================================
// INITIALIZE ESP-NOW
// ============================================================

bool initESPNow() {

    Serial.println();
    Serial.println("========================================");
    Serial.println("           ESP-NOW SETUP");
    Serial.println("========================================");

    // Station mode
    WiFi.mode(WIFI_STA);

    WiFi.disconnect();

    delay(100);

    // --------------------------------------------------------
    // MAC ADDRESS
    // --------------------------------------------------------

    Serial.print("[Sensor] MAC Address : ");
    Serial.println(WiFi.macAddress());

    // --------------------------------------------------------
    // FORCE CHANNEL 11
    // --------------------------------------------------------

    esp_err_t channelResult =
        esp_wifi_set_channel(
            ESP_NOW_CHANNEL,
            WIFI_SECOND_CHAN_NONE
        );

    if (channelResult != ESP_OK) {

        Serial.print(
            "[ERROR] Failed to set channel. Code: "
        );

        Serial.println(channelResult);

        return false;
    }

    Serial.print("[ESP-NOW] Channel     : ");
    Serial.println(ESP_NOW_CHANNEL);

    // --------------------------------------------------------
    // INITIALIZE ESP-NOW
    // --------------------------------------------------------

    esp_err_t initResult =
        esp_now_init();

    if (initResult != ESP_OK) {

        Serial.print(
            "[ERROR] ESP-NOW init failed. Code: "
        );

        Serial.println(initResult);

        return false;
    }

    Serial.println(
        "[ESP-NOW] Initialized successfully."
    );

    // --------------------------------------------------------
    // SEND CALLBACK
    // --------------------------------------------------------

    esp_now_register_send_cb(onDataSent);

    // --------------------------------------------------------
    // ADD GATEWAY AS PEER
    // --------------------------------------------------------

    esp_now_peer_info_t peerInfo = {};

    memcpy(
        peerInfo.peer_addr,
        GATEWAY_MAC,
        6
    );

    peerInfo.channel = ESP_NOW_CHANNEL;

    peerInfo.encrypt = false;

    peerInfo.ifidx = WIFI_IF_STA;

    esp_err_t peerResult =
        esp_now_add_peer(&peerInfo);

    if (
        peerResult != ESP_OK &&
        peerResult != ESP_ERR_ESPNOW_EXIST
    ) {

        Serial.print(
            "[ERROR] Failed to add Gateway peer. Code: "
        );

        Serial.println(peerResult);

        return false;
    }

    Serial.println(
        "[ESP-NOW] Gateway peer added."
    );

    return true;
}

// ============================================================
// READ AND SEND SENSOR DATA
// ============================================================

void sendSensorData() {

    // --------------------------------------------------------
    // READ DHT11
    // --------------------------------------------------------

    float temperature =
        dht.readTemperature();

    float humidity =
        dht.readHumidity();

    // --------------------------------------------------------
    // READ LDR
    // --------------------------------------------------------

    uint16_t lightLevel =
        analogRead(LDR_PIN);

    // --------------------------------------------------------
    // VALIDATE DHT
    // --------------------------------------------------------

    bool sensorValid =
        !isnan(temperature) &&
        !isnan(humidity);

    // --------------------------------------------------------
    // PRINT READINGS
    // --------------------------------------------------------

    Serial.println();
    Serial.println("========================================");
    Serial.println("          SENSOR READINGS");
    Serial.println("========================================");

    if (sensorValid) {

        Serial.print("Temperature : ");
        Serial.print(temperature);
        Serial.println(" C");

        Serial.print("Humidity    : ");
        Serial.print(humidity);
        Serial.println(" %");

    } else {

        Serial.println(
            "[ERROR] DHT11 reading failed."
        );
    }

    Serial.print("Light Level : ");
    Serial.println(lightLevel);

    // --------------------------------------------------------
    // CREATE PACKET
    // --------------------------------------------------------

    SensorPacket_t packet = {};

    packet.nodeID = 1;

    packet.temperature = temperature;

    packet.humidity = humidity;

    packet.lightLevel = lightLevel;

    packet.bootCount = bootCount;

    // Gateway can determine actual RSSI if needed
    packet.rssi = 0;

    // Placeholder until battery measurement is added
    packet.batteryPct = 85;

    packet.sensorValid = sensorValid;

    // --------------------------------------------------------
    // SEND
    // --------------------------------------------------------

    sendComplete = false;
    sendSuccess = false;

    Serial.println();
    Serial.println("[ESP-NOW] Sending packet...");

    esp_err_t result =
        esp_now_send(
            GATEWAY_MAC,
            reinterpret_cast<uint8_t *>(&packet),
            sizeof(packet)
        );

    if (result != ESP_OK) {

        Serial.print(
            "[ERROR] esp_now_send failed. Code: "
        );

        Serial.println(result);

        return;
    }

    // --------------------------------------------------------
    // WAIT FOR SEND CALLBACK
    // --------------------------------------------------------

    unsigned long startTime = millis();

    while (
        !sendComplete &&
        millis() - startTime < 1000
    ) {

        delay(1);
    }

    // --------------------------------------------------------
    // RESULT
    // --------------------------------------------------------

    if (!sendComplete) {

        Serial.println(
            "[ESP-NOW] SEND TIMEOUT"
        );

    }
    else if (sendSuccess) {

        Serial.println(
            "[ESP-NOW] SEND SUCCESS"
        );

    }
    else {

        Serial.println(
            "[ESP-NOW] SEND FAILED"
        );
    }
}

// ============================================================
// DEEP SLEEP
// ============================================================

void goToSleep() {

    Serial.println();
    Serial.println("========================================");
    Serial.println("          POWER MANAGEMENT");
    Serial.println("========================================");

    // Shut down ESP-NOW
    esp_now_deinit();

    // Turn Wi-Fi off
    WiFi.disconnect(true);
    WiFi.mode(WIFI_OFF);

    delay(100);

    Serial.print("[POWER] Sleeping for ");
    Serial.print(SLEEP_SECONDS);
    Serial.println(" seconds.");

    Serial.println(
        "[POWER] ESP32 will wake automatically."
    );

    Serial.flush();

    // --------------------------------------------------------
    // TIMER WAKEUP
    // --------------------------------------------------------

    esp_sleep_enable_timer_wakeup(
        (uint64_t)SLEEP_SECONDS * 1000000ULL
    );

    // --------------------------------------------------------
    // DEEP SLEEP
    // --------------------------------------------------------

    esp_deep_sleep_start();
}

// ============================================================
// SETUP
// ============================================================

void setup() {

    Serial.begin(115200);

    delay(500);

    bootCount++;

    Serial.println();
    Serial.println("========================================");
    Serial.println("          ESP32 SENSOR NODE");
    Serial.println("========================================");

    Serial.print("[POWER] Wake count: ");
    Serial.println(bootCount);

    // --------------------------------------------------------
    // INITIALIZE SENSORS
    // --------------------------------------------------------

    dht.begin();

    pinMode(
        LDR_PIN,
        INPUT
    );

    // DHT11 startup time
    delay(2000);

    Serial.println(
        "[Sensor] DHT11 initialized."
    );

    Serial.println(
        "[Sensor] LDR initialized."
    );

    // --------------------------------------------------------
    // INITIALIZE ESP-NOW
    // --------------------------------------------------------

    if (!initESPNow()) {

        Serial.println();
        Serial.println(
            "ESP-NOW INITIALIZATION FAILED."
        );

        // Retry after next wake
        goToSleep();
    }

    // --------------------------------------------------------
    // SEND ONE PACKET
    // --------------------------------------------------------

    sendSensorData();

    delay(100);

    // --------------------------------------------------------
    // SLEEP
    // --------------------------------------------------------

    goToSleep();
}

// ============================================================
// LOOP
// ============================================================

void loop() {

    // Never reached during normal operation.
    //
    // ESP32 wakes -> setup()
    // sends data -> deep sleep
    // wakes -> setup() again
}
