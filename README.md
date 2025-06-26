#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <esp_sleep.h>

// Chân điều khiển
#define MOTOR_IN1 12
#define MOTOR_IN2 14
#define MOTOR_ENA 13
#define HEAT_IN3 26
#define HEAT_IN4 25

// Chân nút
#define BUTTON_ONOFF 15
#define BUTTON_REVERSE 19
#define BUTTON_SPEED 18
#define BUTTON_HEAT 5

#define PWM_FREQ 21000
#define PWM_RESOLUTION 8
#define MOTOR_PWM_CHANNEL 0

const int SPEED_LEVELS[] = {255, 220, 200};
const int NUM_SPEEDS = 3;

#define DEBOUNCE_DELAY 50

bool systemOn = false;
bool isClockwise = true;
bool isHeatOn = false;
int speedIndex = 0;

struct Button {
  int pin;
  bool lastState;
  bool currentState;
  unsigned long lastDebounce;
};
Button buttons[] = {
  {BUTTON_ONOFF, HIGH, HIGH, 0},
  {BUTTON_REVERSE, HIGH, HIGH, 0},
  {BUTTON_SPEED, HIGH, HIGH, 0},
  {BUTTON_HEAT, HIGH, HIGH, 0}
};

const char* ssid = "ESP32_Motor";
const char* password = "12345678";
WebServer server(80);

void print_wakeup_reason() {
  esp_sleep_wakeup_cause_t reason = esp_sleep_get_wakeup_cause();
  switch (reason) {
    case ESP_SLEEP_WAKEUP_EXT0:
      Serial.println("=> Đánh thức bởi nút nhấn (EXT0)");
      break;
    case ESP_SLEEP_WAKEUP_UNDEFINED:
      Serial.println("=> Bật nguồn hoặc reset thủ công");
      break;
    default:
      Serial.printf("=> Lý do đánh thức khác: %d\n", reason);
      break;
  }
}

void startSystem() {
  resetSystem();
  systemOn = true;
  isClockwise = true;
  speedIndex = 0;

  WiFi.softAP(ssid, password);
  Serial.print("Địa chỉ IP AP: ");
  Serial.println(WiFi.softAPIP());

  server.on("/", handleRoot);
  server.on("/onoff", handleOnOff);
  server.on("/reverse", handleReverse);
  server.on("/speed", handleSpeed);
  server.on("/heat", handleHeat);
  server.begin();
  Serial.println("Máy chủ web đã khởi động");
  setMotor();
  Serial.println("Hệ thống BẬT");
}

void resetSystem() {
  systemOn = false;
  isClockwise = true;
  isHeatOn = false;
  speedIndex = 0;

  digitalWrite(MOTOR_IN1, LOW);
  digitalWrite(MOTOR_IN2, LOW);
  ledcWrite(MOTOR_PWM_CHANNEL, 0);
  digitalWrite(HEAT_IN3, LOW);
  digitalWrite(HEAT_IN4, LOW);
  server.stop();
  WiFi.softAPdisconnect(true);
  Serial.println("Wi-Fi và máy chủ web đã tắt");
  Serial.println("Hệ thống TắT");
}

void setMotor() {
  if (!systemOn) {
    digitalWrite(MOTOR_IN1, LOW);
    digitalWrite(MOTOR_IN2, LOW);
    ledcWrite(MOTOR_PWM_CHANNEL, 0);
    Serial.println("Động cơ TắT");
    return;
  }
  digitalWrite(MOTOR_IN1, isClockwise ? HIGH : LOW);
  digitalWrite(MOTOR_IN2, isClockwise ? LOW : HIGH);
  ledcWrite(MOTOR_PWM_CHANNEL, SPEED_LEVELS[speedIndex]);
  Serial.printf("PWM động cơ: %d\n", SPEED_LEVELS[speedIndex]);
}

void toggleHeat() {
  if (!systemOn) return;
  isHeatOn = !isHeatOn;
  digitalWrite(HEAT_IN3, isHeatOn ? HIGH : LOW);
  digitalWrite(HEAT_IN4, LOW);
  Serial.println(isHeatOn ? "Nhiệt BẬT" : "Nhiệt TắT");
}

void handleButton(int i) {
  bool reading = digitalRead(buttons[i].pin);
  if (reading != buttons[i].lastState) {
    buttons[i].lastDebounce = millis();
  }
  if ((millis() - buttons[i].lastDebounce) > DEBOUNCE_DELAY) {
    if (reading != buttons[i].currentState) {
      buttons[i].currentState = reading;
      if (reading == LOW) {
        switch (i) {
          case 1:
            if (systemOn) {
              isClockwise = !isClockwise;
              Serial.println("Đảo chiều động cơ");
              setMotor();
            }
            break;
          case 2:
            if (systemOn) {
              speedIndex = (speedIndex + 1) % NUM_SPEEDS;
              Serial.printf("Mức tốc độ: %d\n", speedIndex);
              setMotor();
            }
            break;
          case 3:
            toggleHeat();
            break;
        }
      }
    }
  }
  buttons[i].lastState = reading;
}

String getHTML() {
  String html = "<!DOCTYPE html><html lang='vi'><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width,initial-scale=1'>";
  html += "<title>Điều khiển động cơ</title><style>body{font-family:Arial;text-align:center;margin:10px}";
  html += "h3{margin:10px}.button{padding:8px 16px;margin:5px;font-size:14px;border:none;border-radius:3px;cursor:pointer}";
  html += ".onoff{background:#4CAF50;color:#fff}.reverse{background:#2196F3;color:#fff}";
  html += ".speed{background:#FF9800;color:#fff}.heat{background:#F44336;color:#fff}";
  html += ".button:active{opacity:0.7}p{font-size:12px}</style></head><body>";
  html += "<h3>Điều khiển động cơ</h3>";
  html += "<p>Hệ thống: " + String(systemOn ? "BẬT" : "TắT") + "</p>";
  html += "<p>Hướng: " + String(isClockwise ? "Thuận" : "Nghịch") + "</p>";
  html += "<p>Tốc độ: " + String((SPEED_LEVELS[speedIndex] * 100) / 255) + "%</p>";
  html += "<p>Nhiệt: " + String(isHeatOn ? "BẬT" : "TắT") + "</p>";
  html += "<a href='/onoff'><button class='button onoff'>" + String(systemOn ? "TắT" : "BẮT") + "</button></a>";
  html += "<a href='/reverse'><button class='button reverse'>Đảo chiều</button></a>";
  html += "<a href='/speed'><button class='button speed'>Tốc độ</button></a>";
  html += "<a href='/heat'><button class='button heat'>Nhiệt</button></a>";
  html += "</body></html>";
  return html;
}

void handleRoot() { server.send(200, "text/html", getHTML()); }
void handleOnOff() { if (systemOn) resetSystem(); else startSystem(); server.send(200, "text/html", getHTML()); }
void handleReverse() { if (systemOn) { isClockwise = !isClockwise; setMotor(); } server.send(200, "text/html", getHTML()); }
void handleSpeed() { if (systemOn) { speedIndex = (speedIndex + 1) % NUM_SPEEDS; setMotor(); } server.send(200, "text/html", getHTML()); }
void handleHeat() { toggleHeat(); server.send(200, "text/html", getHTML()); }

void setup() {
  Serial.begin(115200);
  pinMode(MOTOR_IN1, OUTPUT);
  pinMode(MOTOR_IN2, OUTPUT);
  pinMode(MOTOR_ENA, OUTPUT);
  pinMode(HEAT_IN3, OUTPUT);
  pinMode(HEAT_IN4, OUTPUT);
  for (int i = 0; i < 4; i++) pinMode(buttons[i].pin, INPUT_PULLUP);
  ledcSetup(MOTOR_PWM_CHANNEL, PWM_FREQ, PWM_RESOLUTION);
  ledcAttachPin(MOTOR_ENA, MOTOR_PWM_CHANNEL);
  print_wakeup_reason();

  if (esp_sleep_get_wakeup_cause() == ESP_SLEEP_WAKEUP_EXT0 && digitalRead(BUTTON_ONOFF) == LOW) {
    Serial.println("=> ESP32 thức dậy do nhấn nút.");
    while (digitalRead(BUTTON_ONOFF) == LOW) delay(10);
    startSystem();
  } else {
    Serial.println("=> Nhấn nút D15 để bật hệ thống.");
  }
}

void loop() {
  if (systemOn) {
    server.handleClient();
    for (int i = 1; i < 4; i++) handleButton(i);
    if (digitalRead(BUTTON_ONOFF) == LOW) {
      Serial.println("=> Nhấn nút lần 2 để vào sleep...");
      while (digitalRead(BUTTON_ONOFF) == LOW) delay(10);
      delay(300);
      resetSystem();
      esp_sleep_enable_ext0_wakeup((gpio_num_t)BUTTON_ONOFF, 0);

      Serial.println("=> Vào chế độ deep sleep...");
      delay(1000);
      esp_deep_sleep_start();
    }
  } else {
    if (digitalRead(BUTTON_ONOFF) == LOW) {
      Serial.println("=> Nhấn để bật lại...");
      while (digitalRead(BUTTON_ONOFF) == LOW) delay(10);
      delay(300);
      startSystem();
    }
  }
  delay(10);
}

