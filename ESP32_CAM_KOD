#include "esp_camera.h"
#include <WiFi.h>
#include <WebServer.h>

#include <Firebase_ESP_Client.h>
#include <HTTPClient.h>
#include "addons/TokenHelper.h"
#include "addons/RTDBHelper.h"


#define WIFI_SSID     "username"
#define WIFI_PASSWORD "password"


#define API_KEY "AIzaSyB54ybTlb8cqaHvkNDsSIzRcZIAu32h5kU"
#define DATABASE_URL "https://bitirmecalismasi-399df-default-rtdb.firebaseio.com/"


#define SENSOR_PIN     13
#define FLASH_LED_PIN  4

// AI Thinker pins
#define PWDN_GPIO_NUM 32
#define RESET_GPIO_NUM -1
#define XCLK_GPIO_NUM 0
#define SIOD_GPIO_NUM 26
#define SIOC_GPIO_NUM 27
#define Y9_GPIO_NUM 35
#define Y8_GPIO_NUM 34
#define Y7_GPIO_NUM 39
#define Y6_GPIO_NUM 36
#define Y5_GPIO_NUM 21
#define Y4_GPIO_NUM 19
#define Y3_GPIO_NUM 18
#define Y2_GPIO_NUM 5
#define VSYNC_GPIO_NUM 25
#define HREF_GPIO_NUM 23
#define PCLK_GPIO_NUM 22

String lastResult = "Hazır (Atık bekleniyor...)";
String machineStatus = "idle";
bool isActive = false;
long lastSize = 0;

WebServer server(80);

long lastDetectionTime = 0;
int lastBrightness = 0;
const int motionThreshold = 8; 


FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

void blinkLed(int t, int d) {
  for (int i = 0; i < t; i++) {
    digitalWrite(FLASH_LED_PIN, HIGH);
    delay(d);
    digitalWrite(FLASH_LED_PIN, LOW);
    delay(d);
  }
}


void syncStatus() {
  WiFiClient client;
  HTTPClient http;
  http.begin(client, "http://192.168.17.8:5000/status");
  int httpCode = http.GET();
  if (httpCode > 0) {
    String status = http.getString();
    status.trim();
    bool oldStatus = isActive;
    isActive = (status == "active");
    if (isActive != oldStatus) {
      Serial.print("\n🔔 Durum Değişti: ");
      Serial.println(isActive ? "AKTİF (Hareket aranıyor)" : "BOŞTA");
    }
  }
  http.end();
}

void checkAndProcessMotion(camera_fb_t* fb) {
  if (millis() - lastDetectionTime < 5000) return; 

  long currentBrightness = 0;
  int sampleCount = 0;
  for (size_t i = 0; i < fb->len; i += 150) {
    currentBrightness += fb->buf[i];
    sampleCount++;
  }
  int avgBrightness = currentBrightness / sampleCount;
  long sizeDiff = abs((long)fb->len - lastSize);
  

  if (lastBrightness > 0) {
    int brightnessDiff = abs(avgBrightness - lastBrightness);
    if (brightnessDiff > 4 || sizeDiff > 1500) {
      Serial.print("\n🔥 [OTOMATİK] Hareket: "); Serial.print(brightnessDiff);
      Serial.print(" | B: "); Serial.println(sizeDiff);
      processRecycling();
      lastDetectionTime = millis();
    }
  }


  
  lastBrightness = avgBrightness;
  lastSize = fb->len;
}

void handleStream() {
  WiFiClient client = server.client();
  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: multipart/x-mixed-replace; boundary=frame");
  client.println();

  while (client.connected()) {
    server.handleClient();
    
    // --- DURUM SENKRONİZASYONU ---
    static unsigned long lastSync = 0;
    if (millis() - lastSync > 3000) {
      syncStatus();
      lastSync = millis();
    }
    
    camera_fb_t* fb = esp_camera_fb_get();
    if (!fb) break;

    // OTOMATİK ALGILAMA (Optimize Edildi)
    static int frameCounter = 0;

    // isActive->kullanıcının qr okutup okutmama durumu ve
    if (isActive && (++frameCounter % 4 == 0)) {
        checkAndProcessMotion(fb);
    }
    // -------------------------

    client.println("--frame");
    client.println("Content-Type: image/jpeg");
    client.println("Content-Length: " + String(fb->len));
    client.println();
    client.write(fb->buf, fb->len);
    client.println();

    esp_camera_fb_return(fb);
    delay(1); // WiFi stack için küçük bir nefes payı
  }
}

/* =======================
   CAPTURE
   ======================= */
void handleCapture() {
  WiFiClient client = server.client();
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) {
    server.send(500, "text/plain", "Camera error");
    return;
  }

  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: image/jpeg");
  client.println("Content-Length: " + String(fb->len));
  client.println();
  client.write(fb->buf, fb->len);

  esp_camera_fb_return(fb);
}


void processRecycling() {
  Serial.println("\n📸 Görüntü alınıyor...");
  
  camera_fb_t * fb = esp_camera_fb_get();
  if(!fb) {
    Serial.println("❌ Kamera hatası!");
    return;
  }

  Serial.println("🚀 Backend'e gönderiliyor (192.168.17.8)...");
  
  WiFiClient client;
  HTTPClient http;
  
  // Flask Backend URL
  http.begin(client, "http://192.168.17.8:5000/predict");
  http.addHeader("Content-Type", "image/jpeg");
  
  // Görüntüyü POST ile gönder
  int httpResponseCode = http.POST(fb->buf, fb->len);
  
  if (httpResponseCode > 0) {
    String response = http.getString();
    Serial.println("✅ Sunucu Yanıtı: " + response);
    
    // Basit bir şekilde türü ayıkla (JSON kütüphanesiz)
    int start = response.indexOf("\"class\":\"") + 9;
    if (start > 9) {
      int end = response.indexOf("\"", start);
      lastResult = "✨ Algılanan: " + response.substring(start, end);
    }
    
    blinkLed(2, 100); 
  } else {
    Serial.print("❌ Gönderim Hatası: ");
    Serial.println(httpResponseCode);
    blinkLed(5, 50); // Hata yanıp sönme (hızlı)
  }
  
  http.end();
  esp_camera_fb_return(fb);
}

void finalizeRecycling() {
  Serial.println("\n💰 Puan aktarma isteği alındı...");
  
  WiFiClient client;
  HTTPClient http;
  
  // Flask /transfer endpoint
  http.begin(client, "http://192.168.17.8:5000/transfer");
  
  int httpResponseCode = http.POST(""); // Boş POST
  
  if (httpResponseCode > 0) {
    String response = http.getString();
    Serial.println("✅ Aktarım Yanıtı: " + response);
    server.send(200, "text/plain", response);
    blinkLed(3, 150);
  } else {
    Serial.print("❌ Aktarım Hatası: ");
    Serial.println(httpResponseCode);
    server.send(500, "text/plain", "Hata: " + String(httpResponseCode));
  }
  http.end();
}

/* =======================
   ROOT
   ======================= */
void handleRoot() {
  String html = "<!DOCTYPE html><html><head>";
  html += "<meta charset='UTF-8'>";
  html += "<meta name='viewport' content='width=device-width, initial-scale=1.0'>";
  html += "<title>ESP32-CAM Geri Dönüşüm</title>";
  html += "<style>body{font-family:Arial;text-align:center;margin:20px;}";
  html += "button{padding:15px 30px;font-size:18px;background:#4CAF50;color:white;border:none;border-radius:5px;cursor:pointer;}";
  html += "button:hover{background:#45a049;}</style>";
  html += "</head><body>";
  html += "<h2>ESP32-CAM Geri Dönüşüm</h2>";
  html += "<img src='/stream' width='100%' style='max-width:640px;'><br><br>";
  html += "<div style='display:flex; justify-content:center; gap:10px;'>";
  html += "  <button onclick='tetikle()' style='background:#4CAF50;'>♻️ Tetikle</button>";
  html += "  <button onclick='puaniAktar()' style='background:#2196F3;'>💰 Puanı Aktar</button>";
  html += "</div>";
  html += "<p id='sonuc'></p>";
  html += "<script>";
  html += "function tetikle(){";
  html += "  document.getElementById('sonuc').innerHTML='İşlem yapılıyor...';";
  html += "  fetch('/recycle?t=' + Date.now()).then(r=>r.text()).then(d=>{";
  html += "    document.getElementById('sonuc').innerHTML='✅ Başarılı! '+d;";
  html += "  }).catch(e=>{";
  html += "    document.getElementById('sonuc').innerHTML='❌ Hata: '+e;";
  html += "  });";
  html += "}";
  html += "function puaniAktar(){";
  html += "  document.getElementById('sonuc').innerHTML='Puan aktarılıyor...';";
  html += "  fetch('/finalize?t=' + Date.now()).then(r=>r.text()).then(d=>{";
  html += "    if (d.includes('success')) {";
  html += "      document.getElementById('sonuc').innerHTML='🎉 Puan Başarıyla Aktarıldı!';";
  html += "      setTimeout(()=> location.reload(), 2000);";
  html += "    } else {";
  html += "      document.getElementById('sonuc').innerHTML='⚠️ '+d;";
  html += "    }";
  html += "  }).catch(e=>{";
  html += "    document.getElementById('sonuc').innerHTML='❌ Hata: '+e;";
  html += "  });";
  html += "}";
  html += "setInterval(() => {";
  html += "  fetch('/info').then(r=>r.text()).then(d=>{";
  html += "    document.getElementById('sonuc').innerHTML = d;";
  html += "  });";
  html += "}, 1500);"; // Her 1.5 saniyede bir yazıyı güncelle
  html += "</script></body></html>";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);

  pinMode(FLASH_LED_PIN, OUTPUT);
  pinMode(SENSOR_PIN, INPUT_PULLUP);

  camera_config_t c;
  c.ledc_channel = LEDC_CHANNEL_0;
  c.ledc_timer = LEDC_TIMER_0;
  c.pin_d0 = Y2_GPIO_NUM;
  c.pin_d1 = Y3_GPIO_NUM;
  c.pin_d2 = Y4_GPIO_NUM;
  c.pin_d3 = Y5_GPIO_NUM;
  c.pin_d4 = Y6_GPIO_NUM;
  c.pin_d5 = Y7_GPIO_NUM;
  c.pin_d6 = Y8_GPIO_NUM;
  c.pin_d7 = Y9_GPIO_NUM;
  c.pin_xclk = XCLK_GPIO_NUM;
  c.pin_pclk = PCLK_GPIO_NUM;
  c.pin_vsync = VSYNC_GPIO_NUM;
  c.pin_href = HREF_GPIO_NUM;
  c.pin_sscb_sda = SIOD_GPIO_NUM;
  c.pin_sscb_scl = SIOC_GPIO_NUM;
  c.pin_pwdn = PWDN_GPIO_NUM;
  c.pin_reset = RESET_GPIO_NUM;
  c.xclk_freq_hz = 20000000;
  c.pixel_format = PIXFORMAT_JPEG;
  c.frame_size = FRAMESIZE_VGA;  // SVGA'dan VGA'ya (640x480) düşürdük, çok daha akıcı olur
  c.jpeg_quality = 15;           // 12'den 15'e (Sayı arttıkça kalite azalıp hız artar)
  c.fb_count = 2;

  esp_camera_init(&c);

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected");
  Serial.print("IP adresi: ");
  Serial.println(WiFi.localIP());

  /* NTP ZAMAN SENKRONİZASYONU */
  Serial.println("\n⏰ Zaman senkronizasyonu yapılıyor...");
  configTime(3 * 3600, 0, "pool.ntp.org", "time.nist.gov");
  
  struct tm timeinfo;
  if (getLocalTime(&timeinfo, 10000)) {
    Serial.println("✅ Zaman senkronize edildi!");
    Serial.print("Tarih: ");
    Serial.println(&timeinfo, "%A, %B %d %Y %H:%M:%S");
  } else {
    Serial.println("⚠️ Zaman senkronizasyonu başarısız (devam ediliyor)");
  }

  /* FIREBASE */
  Serial.println("\n🔥 Firebase bağlantısı kuruluyor...");
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;
  
  // Token status callback (opsiyonel)
  config.token_status_callback = [](TokenInfo info) {
    if (info.status == token_status_ready) {
      Serial.println("✅ Firebase token hazır!");
    }
  };
  
  if (Firebase.signUp(&config, &auth, "", "")) {
    Serial.println("✅ Firebase Auth başarılı!");
  } else {
    Serial.print("❌ Firebase Auth hatası: ");
    Serial.println(config.signer.signupError.message.c_str());
  }
  
  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
  Serial.println("Firebase hazır!\n");

  server.on("/", handleRoot);
  server.on("/stream", handleStream);
  server.on("/capture", handleCapture);
  server.on("/recycle", []() {
    Serial.println("\n🌐 Web isteği alındı: /recycle");
    processRecycling();
    server.send(200, "text/plain", "Tamam");
  });
  server.on("/finalize", finalizeRecycling);
  server.on("/info", []() {
    server.send(200, "text/plain", lastResult);
  });

  server.begin();
}

void loop() {
  server.handleClient();

  if (digitalRead(SENSOR_PIN) == LOW) {
    processRecycling();
    delay(5000);
  }

  static unsigned long lastStatusCheck = 0;
  if (millis() - lastStatusCheck > 3000) {
    syncStatus();
    lastStatusCheck = millis();
  }

  if (isActive) {
    camera_fb_t* fb = esp_camera_fb_get();
    if (fb) {
      checkAndProcessMotion(fb);
      esp_camera_fb_return(fb);
    }
  }
  delay(50);
}
