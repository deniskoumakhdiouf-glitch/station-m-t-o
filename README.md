# Station Météo ESP32

Mesure de température, humidité, pression atmosphérique, qualité de l'air et luminosité. Affichage sur LCD 16x2 et page web WiFi accessible depuis n'importe quel appareil du réseau local.

---

## Matériel

| Composant | Quantité |
|---|---|
| ESP32 WROOM-32 | 1 |
| DHT22 | 1 |
| BMP280 | 1 |
| LCD I2C 16x2 | 1 |
| MQ-135 | 1 |
| LDR | 1 |
| LED RGB cathode commune | 1 |
| Résistance 10kΩ | 2 |
| Résistance 220Ω | 3 |
| Câbles Dupont | ~20 |
| Câble USB | 1 |

---

## Connexions

### DHT22
| DHT22 | ESP32 |
|---|---|
| VCC | 3.3V |
| GND | GND |
| DATA | GPIO4 |

Résistance 10kΩ entre VCC et DATA (pull-up).

### BMP280
| BMP280 | ESP32 |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SDA | GPIO21 |
| SCL | GPIO22 |

### LCD I2C 16x2
| LCD | ESP32 |
|---|---|
| VCC | 5V |
| GND | GND |
| SDA | GPIO21 |
| SCL | GPIO22 |

### MQ-135
| MQ-135 | ESP32 |
|---|---|
| VCC | 5V |
| GND | GND |
| AOUT | GPIO34 |

### LDR
| LDR | ESP32 |
|---|---|
| Patte 1 | 3.3V |
| Patte 2 | GPIO35 |

Résistance 10kΩ entre GPIO35 et GND.

### LED RGB
| LED | ESP32 |
|---|---|
| R | GPIO25 (via 220Ω) |
| G | GPIO26 (via 220Ω) |
| B | GPIO27 (via 220Ω) |
| GND | GND |

---

## Règles importantes

- Ne jamais mettre du 5V sur un GPIO ESP32, il fonctionne en 3.3V
- BMP280 et LCD partagent le même bus I2C (GPIO21/22), c'est normal
- Adresse I2C BMP280 : 0x76
- Adresse I2C LCD : 0x27

---

## Bibliothèques à installer

Dans l'IDE Arduino, menu Outils → Gérer les bibliothèques :

- `DHT sensor library` par Adafruit
- `Adafruit BMP280 Library`
- `LiquidCrystal I2C` par Frank de Brabander

### Ajouter le board ESP32

Fichier → Préférences → URL gestionnaire de cartes :

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Puis Outils → Type de carte → Gestionnaire de cartes → chercher ESP32 → Installer.

---

## Configuration

Ouvrir `station_meteo_esp32.ino` et modifier les deux lignes suivantes :

```cpp
const char* ssid     = "TON_WIFI";
const char* password = "TON_MOT_DE_PASSE";
```

---

## Code

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP280.h>
#include <LiquidCrystal_I2C.h>

const char* ssid     = "TON_WIFI";
const char* password = "TON_MOT_DE_PASSE";

#define PIN_DHT    4
#define PIN_MQ135  34
#define PIN_LDR    35
#define PIN_LED_R  25
#define PIN_LED_G  26
#define PIN_LED_B  27

DHT dht(PIN_DHT, DHT22);
Adafruit_BMP280 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);
WebServer server(80);

float temperature = 0;
float humidite    = 0;
float pression    = 0;
int   qualiteAir  = 0;
int   luminosite  = 0;

void setup() {
  Serial.begin(115200);

  pinMode(PIN_LED_R, OUTPUT);
  pinMode(PIN_LED_G, OUTPUT);
  pinMode(PIN_LED_B, OUTPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Station meteo");
  lcd.setCursor(0, 1);
  lcd.print("Demarrage...");

  dht.begin();

  if (!bmp.begin(0x76)) {
    Serial.println("BMP280 non detecte");
    delay(2000);
  }

  lcd.setCursor(0, 1);
  lcd.print("Connexion WiFi ");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }

  Serial.println("IP : " + WiFi.localIP().toString());
  lcd.setCursor(0, 1);
  lcd.print(WiFi.localIP().toString());
  delay(2000);

  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.begin();
}

void loop() {
  static unsigned long derniereLecture = 0;
  if (millis() - derniereLecture > 2000) {
    lireCapteurs();
    afficherLCD();
    mettreAJourLED();
    derniereLecture = millis();
  }
  server.handleClient();
}

void lireCapteurs() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  if (!isnan(t)) temperature = t;
  if (!isnan(h)) humidite    = h;

  pression   = bmp.readPressure() / 100.0F;
  qualiteAir = analogRead(PIN_MQ135);
  luminosite = analogRead(PIN_LDR);
}

void afficherLCD() {
  lcd.setCursor(0, 0);
  lcd.printf("%.1fC  Hum:%.0f%%  ", temperature, humidite);
  lcd.setCursor(0, 1);
  lcd.printf("%.1f hPa        ", pression);
}

void mettreAJourLED() {
  if (temperature < 18.0) {
    digitalWrite(PIN_LED_R, LOW);
    digitalWrite(PIN_LED_G, LOW);
    digitalWrite(PIN_LED_B, HIGH);
  } else if (temperature <= 28.0) {
    digitalWrite(PIN_LED_R, LOW);
    digitalWrite(PIN_LED_G, HIGH);
    digitalWrite(PIN_LED_B, LOW);
  } else {
    digitalWrite(PIN_LED_R, HIGH);
    digitalWrite(PIN_LED_G, LOW);
    digitalWrite(PIN_LED_B, LOW);
  }
}

// --- Partie générée avec l'aide de l'IA (Claude) ---
// Les fonctions handleRoot() et handleData() ci-dessous ont été
// générées par intelligence artificielle. Le HTML embarqué et la
// structure JSON ont été produits et adaptés avec l'assistance de Claude.
// ------------------------------------------------------------

void handleRoot() {
  String airTexte;
  if      (qualiteAir < 1000) airTexte = "Bonne";
  else if (qualiteAir < 2000) airTexte = "Moyenne";
  else                        airTexte = "Mauvaise";

  int lumPct = map(luminosite, 0, 4095, 0, 100);

  String html = "<!DOCTYPE html><html lang='fr'><head>";
  html += "<meta charset='UTF-8'>";
  html += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  html += "<meta http-equiv='refresh' content='3'>";
  html += "<title>Station meteo</title>";
  html += "<style>";
  html += "body{font-family:sans-serif;background:#1a1a2e;color:#eee;margin:0;padding:20px}";
  html += "h1{text-align:center;color:#e94560}";
  html += ".grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(150px,1fr));gap:16px;margin-top:24px}";
  html += ".card{background:#16213e;border-radius:12px;padding:20px;text-align:center}";
  html += ".val{font-size:2em;font-weight:bold;color:#e94560}";
  html += ".label{font-size:.85em;color:#aaa;margin-top:4px}";
  html += ".footer{text-align:center;margin-top:24px;font-size:.8em;color:#555}";
  html += "</style></head><body>";
  html += "<h1>Station meteo ESP32</h1><div class='grid'>";
  html += "<div class='card'><div class='val'>" + String(temperature, 1) + "C</div><div class='label'>Temperature</div></div>";
  html += "<div class='card'><div class='val'>" + String(humidite, 0)    + "%</div><div class='label'>Humidite</div></div>";
  html += "<div class='card'><div class='val'>" + String(pression, 1)    + " hPa</div><div class='label'>Pression</div></div>";
  html += "<div class='card'><div class='val'>" + airTexte               + "</div><div class='label'>Qualite air</div></div>";
  html += "<div class='card'><div class='val'>" + String(lumPct)         + "%</div><div class='label'>Luminosite</div></div>";
  html += "</div><div class='footer'>Mise a jour toutes les 3 secondes</div></body></html>";

  server.send(200, "text/html", html);
}

void handleData() {
  String json = "{";
  json += "\"temperature\":"  + String(temperature, 1) + ",";
  json += "\"humidite\":"     + String(humidite, 1)    + ",";
  json += "\"pression\":"     + String(pression, 1)    + ",";
  json += "\"qualiteAir\":"   + String(qualiteAir)     + ",";
  json += "\"luminosite\":"   + String(luminosite);
  json += "}";
  server.send(200, "application/json", json);
}
// --- Fin de la partie générée avec l'IA ---
```

---

## Utilisation

1. Flasher le code sur l'ESP32
2. Ouvrir le moniteur série à 115200 bauds
3. Attendre l'affichage de l'adresse IP
4. Ouvrir un navigateur et entrer l'adresse IP
5. Le dashboard s'affiche et se met à jour toutes les 3 secondes

---

## Comportement LED RGB

| Température | Couleur |
|---|---|
| Moins de 18°C | Bleu |
| Entre 18°C et 28°C | Vert |
| Plus de 28°C | Rouge |

---

## Dépannage

| Problème | Cause | Solution |
|---|---|---|
| BMP280 non détecté | Mauvaise adresse I2C | Essayer 0x77 à la place de 0x76 |
| LCD n'affiche rien | Mauvaise adresse I2C | Essayer 0x3F à la place de 0x27 |
| DHT22 renvoie NaN | Mauvais câblage | Vérifier la résistance pull-up 10kΩ |
| Page web inaccessible | WiFi non connecté | Vérifier SSID et mot de passe |
| Valeurs MQ-135 erratiques | Capteur froid | Laisser chauffer 2 minutes |# station-m-t-o
