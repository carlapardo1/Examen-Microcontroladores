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
```

## 2. Inicialización del MKR IoT Carrier y variables globales 
Incluye librerías de sensores, LEDs, buzzer, pantalla y variables para controlar todo el sistema.

```cpp
// =========================================================
//   MKR IOT CARRIER + APDS
// =========================================================
#include <Arduino_MKRIoTCarrier.h>
#include <Arduino_APDS9960.h>

MKRIoTCarrier carrier;

// ====== VARIABLES ======
float temperatura = 0;
bool rojo = false;
bool azul = false;
bool verde = false;

// Control manual del foco (solo desde IoT)
bool focoManualOn = false;

unsigned long lastTempRead = 0;
const unsigned long TEMP_READ_MS = 2000;

unsigned long lastTouchEvent = 0;
const unsigned long TOUCH_DEBOUNCE_MS = 250;

// === Buzzer Touch 3 ===
bool buzzerOnlyActive = false;
unsigned long buzzerOnlyTimer = 0;

// === Secuencia Touch 4 ===
bool sequenceActive = false;
int sequenceStep = 0;
unsigned long sequenceLastStepTime = 0;
const int SEQUENCE_SPEED = 200;
```

## 3. Callbacks de Blynk (control desde la app IoT)
Estas funciones se ejecutan cuando la app modifica un pin virtual. Controlan LEDs y modo del foco.

```cpp
// =========================================================
//                 BLYNK WRITE CALLBACKS
// =========================================================

// ROJO (V1)
BLYNK_WRITE(V1) {
  rojo = param.asInt();
  updateCarrierLeds();
}

// AZUL (V2)
BLYNK_WRITE(V2) {
  azul = param.asInt();
  updateCarrierLeds();
}

// VERDE (V3)
BLYNK_WRITE(V3) {
  verde = param.asInt();
  updateCarrierLeds();
}

// FOCO MANUAL (V4)
BLYNK_WRITE(V4) {
  focoManualOn = param.asInt();
  controlFocoAutomatico();
}
```

## 4. Setup del sistema
Inicializa display, sensores, WiFi y el estado inicial del foco.
```cpp
// =========================================================
//                        SETUP
// =========================================================

void setup() {
  Serial.begin(9600);
  delay(1500);

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  carrier.withCase();
  carrier.begin();
  carrier.display.setRotation(2);

  if (!APDS.begin()) {
    Serial.println("Error al iniciar APDS9960");
  }

  carrier.leds.setBrightness(100);
  carrier.leds.clear();
  carrier.leds.show();

  controlFocoAutomatico();
  updateScreen();
}
```

## 5. Loop principal
Aquí se ejecutan todas las funciones repetitivas del sistema.
```cpp
// =========================================================
//                        LOOP
// =========================================================
void loop() {
  Blynk.run();

  checkTouchButtons();
  checkTemperature();

  if (buzzerOnlyActive) {
    if (millis() - buzzerOnlyTimer >= 1000) {
      buzzerOnlyActive = false;
      carrier.Buzzer.noSound();
    }
  }

  if (sequenceActive) {
    if (millis() - sequenceLastStepTime >= SEQUENCE_SPEED) {
      sequenceLastStepTime = millis();

      if (sequenceStep < 5) {
        carrier.leds.setPixelColor(sequenceStep, 0, 255, 255);
        carrier.leds.show();
        sequenceStep++;
      } else {
        sequenceActive = false;
        updateCarrierLeds();
      }
    }
  }
}
```

## 6. Actualización de LEDs
Controla los LEDs del carrier desde IoT o botones.
```cpp
// =========================================================
//                LEDS – ACTUALIZAR DESDE APP
// =========================================================
void updateCarrierLeds() {
  carrier.leds.clear();

  if (rojo)  carrier.leds.fill(carrier.leds.Color(255, 0, 0), 0, 5);
  if (azul)  carrier.leds.fill(carrier.leds.Color(0, 0, 255), 0, 5);
  if (verde) carrier.leds.fill(carrier.leds.Color(0, 255, 0), 0, 5);

  carrier.leds.show();

  Blynk.virtualWrite(V1, rojo);
  Blynk.virtualWrite(V2, azul);
  Blynk.virtualWrite(V3, verde);

  updateScreen();
}
```

## 7. Lectura de botones touch
Control local del sistema sin necesidad de IoT.
```cpp
// =========================================================
//                     TOUCH BUTTONS
// =========================================================
void checkTouchButtons() {
  carrier.Buttons.update();

  if (millis() - lastTouchEvent < TOUCH_DEBOUNCE_MS) return;

  if (carrier.Buttons.onTouchDown(TOUCH0)) {
    lastTouchEvent = millis();
    rojo = !rojo;
    updateCarrierLeds();
  }

  if (carrier.Buttons.onTouchDown(TOUCH1)) {
    lastTouchEvent = millis();
    azul = !azul;
    updateCarrierLeds();
  }

  if (carrier.Buttons.onTouchDown(TOUCH2)) {
    lastTouchEvent = millis();
    verde = !verde;
    updateCarrierLeds();
  }

  if (carrier.Buttons.onTouchDown(TOUCH3)) {
    lastTouchEvent = millis();
    buzzerOnlyActive = true;
    buzzerOnlyTimer = millis();
    carrier.Buzzer.sound(500);
  }

  if (carrier.Buttons.onTouchDown(TOUCH4)) {
    lastTouchEvent = millis();
    sequenceActive = true;
    sequenceStep = 0;
    sequenceLastStepTime = millis();
    carrier.leds.clear();
    carrier.leds.show();
  }
}
```

## 8. Control automático del foco según temperatura
Si el modo manual está activado, tiene prioridad.
```cpp
// =========================================================
//          CONTROL AUTOMÁTICO DEL FOCO (RELAY)
// =========================================================
void controlFocoAutomatico() {
  if (focoManualOn) {
    carrier.Relay1.open();
    return;
  }

  if (temperatura > 29) {
    carrier.Relay1.open();
  } 
  else if (temperatura <= 28) {
    carrier.Relay1.close();
  }
}
```

## 9. Lectura de temperatura
Se envía a Blynk y controla el foco.
```cpp
// =========================================================
//                   TEMPERATURA
// =========================================================
void checkTemperature() {
  if (millis() - lastTempRead < TEMP_READ_MS) return;
  lastTempRead = millis();

  temperatura = carrier.Env.readTemperature();
  Serial.print("Temp: "); Serial.println(temperatura);

  controlFocoAutomatico();
  Blynk.virtualWrite(V0, temperatura);

  updateScreen();
}
```

## 10. Pantalla TFT
Muestra estado actual del sistema.
```cpp
// =========================================================
//                     PANTALLA TFT
// =========================================================
void updateScreen() {
  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setTextSize(2);
  carrier.display.setTextColor(ST77XX_WHITE);

  carrier.display.setCursor(20, 70);
  carrier.display.print("Temp: ");
  carrier.display.print(temperatura, 1);
  carrier.display.println(" C");

  carrier.display.setCursor(20, 120);
  carrier.display.print("Rojo: ");
  carrier.display.println(rojo ? "ON" : "OFF");

  carrier.display.setCursor(20, 160);
  carrier.display.print("Azul: ");
  carrier.display.println(azul ? "ON" : "OFF");

  carrier.display.setCursor(20, 200);
  carrier.display.print("Verde: ");
  carrier.display.println(verde ? "ON" : "OFF");
}

#Código Completo

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

// =========================================================
//   MKR IOT CARRIER + APDS
// =========================================================
#include <Arduino_MKRIoTCarrier.h>
#include <Arduino_APDS9960.h>

MKRIoTCarrier carrier;

// ====== VARIABLES ======
float temperatura = 0;
bool rojo = false;
bool azul = false;
bool verde = false;

// Control manual del foco (solo desde IoT)
bool focoManualOn = false;

unsigned long lastTempRead = 0;
const unsigned long TEMP_READ_MS = 2000;

// Touch debounce
unsigned long lastTouchEvent = 0;
const unsigned long TOUCH_DEBOUNCE_MS = 250;

// === Buzzer Touch 3 ===
bool buzzerOnlyActive = false;
unsigned long buzzerOnlyTimer = 0;

// === Secuencia Touch 4 ===
bool sequenceActive = false;
int sequenceStep = 0;
unsigned long sequenceLastStepTime = 0;
const int SEQUENCE_SPEED = 200;

// =========================================================
//                 BLYNK WRITE CALLBACKS
// =========================================================

// ROJO (V1)
BLYNK_WRITE(V1) {
  rojo = param.asInt();
  updateCarrierLeds();
}

// AZUL (V2)
BLYNK_WRITE(V2) {
  azul = param.asInt();
  updateCarrierLeds();
}

// VERDE (V3)
BLYNK_WRITE(V3) {
  verde = param.asInt();
  updateCarrierLeds();
}

// FOCO MANUAL (V4) -> SOLO DESDE IOT
BLYNK_WRITE(V4) {
  focoManualOn = param.asInt();  // 0 = auto por temperatura, 1 = encendido manual
  controlFocoAutomatico();       // actualiza el relay de inmediato
}

// =========================================================
//                        SETUP
// =========================================================

void setup() {
  Serial.begin(9600);
  delay(1500);

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  carrier.withCase();
  carrier.begin();
  carrier.display.setRotation(2);

  if (!APDS.begin()) {
    Serial.println("Error al iniciar APDS9960");
  }

  carrier.leds.setBrightness(100);
  carrier.leds.clear();
  carrier.leds.show();

  // Asegurar foco en estado inicial correcto
  controlFocoAutomatico();

  updateScreen();
}

// =========================================================
//                        LOOP
// =========================================================
void loop() {
  Blynk.run();

  checkTouchButtons();
  checkTemperature();

  // === Buzzer Touch 3 (1 segundo) ===
  if (buzzerOnlyActive) {
    if (millis() - buzzerOnlyTimer >= 1000) {
      buzzerOnlyActive = false;
      carrier.Buzzer.noSound();
    }
  }

  // === Secuencia Touch 4 ===
  if (sequenceActive) {
    if (millis() - sequenceLastStepTime >= SEQUENCE_SPEED) {
      sequenceLastStepTime = millis();

      if (sequenceStep < 5) {
        carrier.leds.setPixelColor(sequenceStep, 0, 255, 255); // AQUA
        carrier.leds.show();
        sequenceStep++;
      } else {
        sequenceActive = false;
        updateCarrierLeds();  // Regresa a LEDs normales
      }
    }
  }
}

// =========================================================
//                LEDS – ACTUALIZAR DESDE APP
// =========================================================
void updateCarrierLeds() {
  carrier.leds.clear();

  if (rojo)  carrier.leds.fill(carrier.leds.Color(255, 0, 0), 0, 5);
  if (azul)  carrier.leds.fill(carrier.leds.Color(0, 0, 255), 0, 5);
  if (verde) carrier.leds.fill(carrier.leds.Color(0, 255, 0), 0, 5);

  carrier.leds.show();

  // Actualizar estado en Blynk
  Blynk.virtualWrite(V1, rojo);
  Blynk.virtualWrite(V2, azul);
  Blynk.virtualWrite(V3, verde);

  updateScreen();
}

// =========================================================
//                     TOUCH BUTTONS
// =========================================================
void checkTouchButtons() {
  carrier.Buttons.update();

  if (millis() - lastTouchEvent < TOUCH_DEBOUNCE_MS) return;

  // TOUCH 0 → ROJO
  if (carrier.Buttons.onTouchDown(TOUCH0)) {
    lastTouchEvent = millis();
    rojo = !rojo;
    updateCarrierLeds();
  }

  // TOUCH 1 → AZUL
  if (carrier.Buttons.onTouchDown(TOUCH1)) {
    lastTouchEvent = millis();
    azul = !azul;
    updateCarrierLeds();
  }

  // TOUCH 2 → VERDE
  if (carrier.Buttons.onTouchDown(TOUCH2)) {
    lastTouchEvent = millis();
    verde = !verde;
    updateCarrierLeds();
  }

  // TOUCH 3 → Buzzer 1 segundo
  if (carrier.Buttons.onTouchDown(TOUCH3)) {
    lastTouchEvent = millis();
    buzzerOnlyActive = true;
    buzzerOnlyTimer = millis();
    carrier.Buzzer.sound(500);
  }

  // TOUCH 4 → Secuencia LEDs
  if (carrier.Buttons.onTouchDown(TOUCH4)) {
    lastTouchEvent = millis();
    sequenceActive = true;
    sequenceStep = 0;
    sequenceLastStepTime = millis();
    carrier.leds.clear();
    carrier.leds.show();
  }
}

// =========================================================
//          CONTROL AUTOMÁTICO DEL FOCO (RELAY)
// =========================================================
// Nota:
// - Si focoManualOn == true  -> foco SIEMPRE encendido (prioridad manual)
// - Si focoManualOn == false -> control por temperatura (automático, NO aparece en IoT)
void controlFocoAutomatico() {
  if (focoManualOn) {
    // MODO MANUAL: foco siempre encendido
    carrier.Relay1.open();   // foco ON
    return;
  }

  // MODO AUTOMÁTICO POR TEMPERATURA
  if (temperatura > 29) {
    carrier.Relay1.open();   // foco ON
  } 
  else if (temperatura <= 28) {
    carrier.Relay1.close();  // foco OFF
  }
}

// =========================================================
//                   TEMPERATURA
// =========================================================
void checkTemperature() {
  if (millis() - lastTempRead < TEMP_READ_MS) return;
  lastTempRead = millis();

  temperatura = carrier.Env.readTemperature();
  Serial.print("Temp: "); Serial.println(temperatura);

  // Control del foco (manual tiene prioridad, si no, automático)
  controlFocoAutomatico();

  // Mandar a Blynk
  Blynk.virtualWrite(V0, temperatura);

  updateScreen();
}

// =========================================================
//                     PANTALLA TFT
// =========================================================
void updateScreen() {
  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setTextSize(2);
  carrier.display.setTextColor(ST77XX_WHITE);

  carrier.display.setCursor(20, 70);
  carrier.display.print("Temp: ");
  carrier.display.print(temperatura, 1);
  carrier.display.println(" C");

  carrier.display.setCursor(20, 120);
  carrier.display.print("Rojo: ");
  carrier.display.println(rojo ? "ON" : "OFF");

  carrier.display.setCursor(20, 160);
  carrier.display.print("Azul: ");
  carrier.display.println(azul ? "ON" : "OFF");

  carrier.display.setCursor(20, 200);
  carrier.display.print("Verde: ");
  carrier.display.println(verde ? "ON" : "OFF");

}
```

#Lo queremos profe atoany!!






