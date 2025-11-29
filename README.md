# Examen – Microcontroladores  
Uso de Arduino MKR con IoT (Blynk + MKR IoT Carrier)

## Descripción general  
Este proyecto utiliza un **Arduino MKR WiFi 1010** junto con el **MKR IoT Carrier** y la plataforma **Blynk IoT** para controlar LEDs, buzzer, temperatura, pantalla TFT y un foco con control **automático** y **manual desde IoT**.

Incluye:  
- Control de LEDs RGB desde IoT y botones touch  
- Lectura de temperatura y control automático del foco  
- Buzzer activado por touch  
- Secuencia animada de LEDs  
- Actualización constante en pantalla TFT  
- Comunicación IoT mediante Virtual Pins  

---

# Código utilizado

---

## 1. Configuración de WiFi y Blynk  
En esta parte se inicializa la conexión con la plataforma Blynk y el WiFi del Arduino.

```cpp
// ========================================================= 
//   BLYNK + WIFI
// =========================================================
#define BLYNK_TEMPLATE_ID "TMPL27IWAPzUG"
#define BLYNK_TEMPLATE_NAME "Examen"
#define BLYNK_AUTH_TOKEN "eI8X3Cf_Bd3qUr_WVtO66TVolRFdajtF"

#include <WiFiNINA.h>
#include <BlynkSimpleWiFiNINA.h>

char ssid[] = "CpiPhone";
char pass[] = "123456789";
