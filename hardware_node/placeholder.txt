#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

// --- Configuration ---
#define WIFI_SSID "Redmi Note 10S"
#define WIFI_PASSWORD "kasthuri"
#define API_KEY "AIzaSyAPVXLNeluoFs6SS0-cq5gijXyVln23m8E"
#define DATABASE_URL "project-aegis-fd75b-default-rtdb.asia-southeast1.firebasedatabase.app"

// --- Hardware Setup ---
Adafruit_MPU6050 mpu;
const int MQ_PIN = 34; // Ensure this matches your wiring for the gas sensor

// --- Firebase Objects ---
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10); 
  
  // 1. WiFi Connection
  Serial.print("\nConnecting to Wi-Fi");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(300);
  }
  Serial.println("\nConnected to Wi-Fi!");

  // 2. Hardware Init (Using the proven isolation logic)
  Serial.println("Initializing MPU6050...");
  if (!mpu.begin()) {
    Serial.println("Failed to find MPU6050 chip!");
    while (1) {
      delay(10); // Halt if it fails
    }
  }
  Serial.println("MPU6050 Found! Sensor is ALIVE.");
  
  // Set basic sensor ranges (From your working test)
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  mpu.setGyroRange(MPU6050_RANGE_500_DEG);
  mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
  delay(100);

  // Setup Gas Sensor Pin
  pinMode(MQ_PIN, INPUT);

  // 3. Firebase Config
  Serial.println("Connecting to Firebase...");
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;
  config.signer.test_mode = true;
  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
}

void loop() {
  // Read Gas Data
  int gasValue = analogRead(MQ_PIN);
  
  // Read MPU6050 Data
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  // Send to Firebase
  if (Firebase.ready()) {
    // Create unique paths for the node
    Firebase.RTDB.setInt(&fbdo, "/node1/gas", gasValue);
    Firebase.RTDB.setFloat(&fbdo, "/node1/accelX", a.acceleration.x);
    Firebase.RTDB.setFloat(&fbdo, "/node1/accelY", a.acceleration.y);
    Firebase.RTDB.setFloat(&fbdo, "/node1/accelZ", a.acceleration.z);
    
    // Print to Serial Monitor so you can verify it's working
    Serial.print("Gas: ");
    Serial.print(gasValue);
    Serial.print(" | Accel Z: ");
    Serial.print(a.acceleration.z);
    Serial.println(" | Data synced to Cloud.");
  } else {
    Serial.println("Firebase not ready yet...");
  }
  
  delay(2000); // Send update every 2 seconds
}
