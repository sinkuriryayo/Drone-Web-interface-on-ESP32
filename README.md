# Drone-Web-interface-on-ESP32
This web app is aim to interact with drone using smartphone/Pc via web interface.

#include <WiFi.h>
#include <ESPAsyncWebServer.h>

// AP credentials
const char* ssid = "ESP32-Drone";
const char* password = "123456789";

// PWM setup
const int pwmPin = 23;
const int pwmFreq = 50;
const int pwmChannel = 0;
const int pwmResolution = 16;

// Motor state
bool motorActive = false;

AsyncWebServer server(80);

// Function to write PWM in microseconds
void writeMicroseconds(uint8_t channel, int microseconds) {
  int duty = (microseconds * 65536) / 20000;
  ledcWrite(channel, duty);
}

void setup() {
  Serial.begin(115200);
  WiFi.softAP(ssid, password);
  delay(1000);
  Serial.println("ESP32 Access Point Started");
  Serial.print("Access Point IP: http://");
  Serial.println(WiFi.softAPIP());

  ledcSetup(pwmChannel, pwmFreq, pwmResolution);
  ledcAttachPin(pwmPin, pwmChannel);
  writeMicroseconds(pwmChannel, 1000);
  delay(2000);

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    String html = R"rawliteral(
      <!DOCTYPE html>
      <html>
      <head>
        <title>Quadcopter Control</title>
        <style>
          body {
            font-family: Arial, sans-serif;
            background: #121212;
            color: #f5f5f5;
            text-align: center;
            padding: 20px;
          }
          h2 { color: #03dac6; }
          .slider-container {
            display: flex;
            justify-content: space-around;
            align-items: center;
            margin-top: 30px;
          }
          .vertical-slider {
            writing-mode: bt-lr; /* vertical orientation */
            -webkit-appearance: slider-vertical;
            width: 40px;
            height: 200px;
            margin: 20px;
          }
          .horizontal-slider {
            width: 300px;
            margin-top: 40px;
          }
          .master-button {
            background-color: #03dac6;
            color: #000;
            border: none;
            padding: 15px 30px;
            font-size: 18px;
            cursor: pointer;
            margin-top: 40px;
            border-radius: 10px;
          }
          .master-button:hover {
            background-color: #018786;
          }
        </style>
      </head>
      <body>
        <h2>Quadcopter BLDC Motor Control</h2>

        <div class="slider-container">
          <div>
            <p>Forward / Backward</p>
            <input type="range" min="-100" max="100" value="0" class="vertical-slider" id="sliderFB" oninput="sendControl()">
            <p id="valFB">0</p>
          </div>
          <div>
            <p>Left / Right</p>
            <input type="range" min="-100" max="100" value="0" class="vertical-slider" id="sliderLR" oninput="sendControl()">
            <p id="valLR">0</p>
          </div>
        </div>

        <p>Throttle</p>
        <input type="range" min="0" max="255" value="0" class="horizontal-slider" id="sliderThrottle" oninput="sendControl()">
        <p id="valThrottle">0</p>

        <button class="master-button" onclick="toggleMotor()">Start / Stop Motors</button>

        <script>
          let motorOn = false;

          function sendControl() {
            let fb = document.getElementById("sliderFB").value;
            let lr = document.getElementById("sliderLR").value;
            let throttle = document.getElementById("sliderThrottle").value;

            document.getElementById("valFB").innerText = fb;
            document.getElementById("valLR").innerText = lr;
            document.getElementById("valThrottle").innerText = throttle;

            fetch(`/control?fb=${fb}&lr=${lr}&throttle=${throttle}`);
          }

          function toggleMotor() {
            motorOn = !motorOn;
            fetch(`/motor?state=${motorOn ? "on" : "off"}`);
          }
        </script>
      </body>
      </html>
    )rawliteral";
    request->send(200, "text/html", html);
  });

  // Handle motor start/stop
  server.on("/motor", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasParam("state")) {
      String state = request->getParam("state")->value();
      motorActive = (state == "on");
      Serial.print("Motor state changed to: ");
      Serial.println(motorActive ? "ON" : "OFF");

      if (!motorActive) {
        writeMicroseconds(pwmChannel, 1000);  // Send stop signal
      }
    }
    request->send(200, "text/plain", "OK");
  });

  // Handle slider control
  server.on("/control", HTTP_GET, [](AsyncWebServerRequest *request){
    if (!motorActive) {
      request->send(200, "text/plain", "Motor off");
      return;
    }

    int fb = request->getParam("fb")->value().toInt();
    int lr = request->getParam("lr")->value().toInt();
    int throttle = request->getParam("throttle")->value().toInt();

    Serial.printf("FB: %d | LR: %d | Throttle: %d\n", fb, lr, throttle);

    // For now, only throttle affects PWM
    int microseconds = map(throttle, 0, 255, 1000, 2000);
    writeMicroseconds(pwmChannel, microseconds);

    request->send(200, "text/plain", "Control updated");
  });

  server.begin();
}

void loop() {
  // Nothing to do here
}

