# edu-edge-net

## WIF

```c
#include "esp_wifi.h"
#include "esp_netif.h"
#include "lwip/lwip_napt.h"
#include <WiFi.h>

const char* STA_SSID = "SSID-connecting-router";
const char* STA_PASS = "PASSWORD-connecting-router";

const char* AP_SSID  = "IOT";
//const char* AP_PASS  = "1234567890"; //Encryption
const char* AP_PASS  = ""; //No encryption

void setup() {
  Serial.begin(115200);
  delay(1000);

  WiFi.mode(WIFI_AP_STA);

  // AP first
  //1 — channel
  //0 — SSID not hidden
  //10 — max connections
  WiFi.softAP(AP_SSID, AP_PASS, 1, 0, 10);
  Serial.println("AP started");
  Serial.print("AP SSID: ");
  Serial.println(WiFi.softAPSSID());
  Serial.print("AP IP: ");
  Serial.println(WiFi.softAPIP());
  

  // Then connect to router
  WiFi.begin(STA_SSID, STA_PASS);
  Serial.print("Connecting to router");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected to router");
  Serial.print("Connected to: ");
  Serial.println(WiFi.SSID());

  // Enable NAT
  // esp_netif_napt_enable(esp_netif_get_handle_from_ifkey("WIFI_AP_DEF"));

  Serial.print("Extender IP: ");
  Serial.println(WiFi.softAPIP());
}

void loop() {
  // Nothing needed
}
```

