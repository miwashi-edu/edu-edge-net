# edu-edge-net

## WIFI Router

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

## Wifi Client

```python
import network
import socket
import time
from machine import Pin

# ── Configuration ──────────────────────────────────────────────
SSID     = "IOT"       # Replace with your router's SSID
PASSWORD = "1234567890"   # Replace with your Wi-Fi password

BUTTON_PIN       = 14    # GP14 — connect / disconnect
HELLO_BUTTON_PIN = 15    # GP15 — send "hello" to router
LED_PIN          = "LED" # Onboard LED (use a pin number if using external LED)

ROUTER_PORT    = 8080  # TCP port the router listens on
HELLO_MSG      = "hello"

DEBOUNCE_MS     = 200  # Milliseconds to debounce the buttons
CONNECT_TIMEOUT = 15   # Seconds to wait for connection before giving up
# ───────────────────────────────────────────────────────────────

# Hardware setup
button       = Pin(BUTTON_PIN,       Pin.IN, Pin.PULL_UP)  # Button between GP14 and GND
hello_button = Pin(HELLO_BUTTON_PIN, Pin.IN, Pin.PULL_UP)  # Button between GP15 and GND
led          = Pin(LED_PIN, Pin.OUT)

# Wi-Fi interface
wlan = network.WLAN(network.STA_IF)

# ── LED helpers ────────────────────────────────────────────────
def led_on():
    led.value(1)

def led_off():
    led.value(0)

def led_blink(times=3, delay_ms=150):
    """Blink the LED `times` times."""
    for _ in range(times):
        led.value(1)
        time.sleep_ms(delay_ms)
        led.value(0)
        time.sleep_ms(delay_ms)

# ── Wi-Fi helpers ──────────────────────────────────────────────
def connect():
    """Activate the interface and connect to the router."""
    print("Connecting to", SSID, "...")
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)

    deadline = time.ticks_add(time.ticks_ms(), CONNECT_TIMEOUT * 1000)
    while not wlan.isconnected():
        if time.ticks_diff(deadline, time.ticks_ms()) <= 0:
            print("Connection timed out.")
            led_blink(times=6, delay_ms=80)   # fast blinks = failure
            wlan.active(False)
            return False
        led.toggle()          # slow blink while connecting
        time.sleep_ms(300)

    led_on()                  # solid LED = connected
    ip = wlan.ifconfig()[0]
    print("Connected! IP:", ip)
    return True

def disconnect():
    """Disconnect and deactivate the interface."""
    print("Disconnecting ...")
    wlan.disconnect()
    wlan.active(False)
    led_off()
    print("Disconnected.")

def is_connected():
    return wlan.active() and wlan.isconnected()

def send_hello():
    """Open a TCP connection to the router's gateway and send 'hello'."""
    if not is_connected():
        print("Not connected — cannot send hello.")
        led_blink(times=3, delay_ms=80)
        return
    gateway = wlan.ifconfig()[2]   # default gateway = router IP
    print("Sending '{}' to {}:{}".format(HELLO_MSG, gateway, ROUTER_PORT))
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(5)
        s.connect((gateway, ROUTER_PORT))
        s.send(HELLO_MSG.encode())
        print("Sent successfully.")
        s.close()
        led_blink(times=2, delay_ms=100)   # 2 quick blinks = message sent
        led_on()                           # back to solid = still connected
    except Exception as e:
        print("TCP send failed:", e)
        led_blink(times=4, delay_ms=100)   # 4 blinks = send error
        led_on()

# ── Main loop ──────────────────────────────────────────────────
print("Pico 2 W Wi-Fi toggle ready.")
print("GP14 = connect / disconnect | GP15 = send hello")

last_press_ms       = 0   # debounce for GP14
last_hello_press_ms = 0   # debounce for GP15

while True:
    # ── GP14: connect / disconnect ─────────────────────────────
    if button.value() == 0:
        now = time.ticks_ms()
        if time.ticks_diff(now, last_press_ms) > DEBOUNCE_MS:
            last_press_ms = now

            if is_connected():
                disconnect()
            else:
                connect()

            # Wait for button release before continuing
            while button.value() == 0:
                time.sleep_ms(10)

    # ── GP15: send hello ───────────────────────────────────────
    if hello_button.value() == 0:
        now = time.ticks_ms()
        if time.ticks_diff(now, last_hello_press_ms) > DEBOUNCE_MS:
            last_hello_press_ms = now

            send_hello()

            # Wait for button release before continuing
            while hello_button.value() == 0:
                time.sleep_ms(10)

    time.sleep_ms(20)   # small sleep keeps the CPU happy
```



