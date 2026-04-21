# edu-edge-net

## WIF

```c
#include "esp_wifi.h"
#include "esp_netif.h"
#include "lwip/lwip_napt.h"
#include <WiFi.h>

const char* STA_SSID = "SSID";
const char* STA_PASS = "PASSWORD";

const char* AP_SSID  = "IOT";
const char* AP_PASS  = "1234567890";

void wifiEvent(WiFiEvent_t event, WiFiEventInfo_t info) {
  switch(event) {

    case ARDUINO_EVENT_WIFI_AP_STACONNECTED:
      Serial.print("Device connected, MAC: ");
      Serial.printf("%02X:%02X:%02X:%02X:%02X:%02X\n",
        info.wifi_ap_staconnected.mac[0],
        info.wifi_ap_staconnected.mac[1],
        info.wifi_ap_staconnected.mac[2],
        info.wifi_ap_staconnected.mac[3],
        info.wifi_ap_staconnected.mac[4],
        info.wifi_ap_staconnected.mac[5]);
      Serial.print("Total connected: ");
      Serial.println(WiFi.softAPgetStationNum());
      break;

    case ARDUINO_EVENT_WIFI_AP_STADISCONNECTED:
      Serial.print("Device disconnected, MAC: ");
      Serial.printf("%02X:%02X:%02X:%02X:%02X:%02X\n",
        info.wifi_ap_stadisconnected.mac[0],
        info.wifi_ap_stadisconnected.mac[1],
        info.wifi_ap_stadisconnected.mac[2],
        info.wifi_ap_stadisconnected.mac[3],
        info.wifi_ap_stadisconnected.mac[4],
        info.wifi_ap_stadisconnected.mac[5]);
      Serial.print("Total connected: ");
      Serial.println(WiFi.softAPgetStationNum());
      break;

    case ARDUINO_EVENT_WIFI_STA_DISCONNECTED:
      Serial.println("Lost connection to router — reconnecting...");
      WiFi.begin(STA_SSID, STA_PASS);
      break;
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  WiFi.onEvent(wifiEvent);

  WiFi.persistent(false);
  WiFi.mode(WIFI_OFF);
  delay(100);
  WiFi.mode(WIFI_AP_STA);

  // AP first
  WiFi.softAP(AP_SSID, AP_PASS);
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
  Serial.print("Signal strength: ");
  Serial.print(WiFi.RSSI());
  Serial.println(" dBm");
  Serial.print("Channel: ");
  Serial.println(WiFi.channel());

  // Enable NAT
  // esp_netif_napt_enable(esp_netif_get_handle_from_ifkey("WIFI_AP_DEF"));
}

void loop() {
  
}
```

