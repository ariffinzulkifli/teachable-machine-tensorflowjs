# Teachable Machine TensoflowJS

Real-time machine learning inference dashboards that connect Google's Teachable Machine models to IoT devices via MQTT. Train your models in the browser, run inference on the web, and control hardware like the [Myduino AIoT Education Kit](https://myduino.com/product/myduino-aiot-education-kit/).

## Overview

This project provides three web-based dashboards for different types of machine learning models:

- **Image Classification** - Classify objects, gestures, or scenes using your webcam
- **Audio Classification** - Classify sounds, speech, or music using your microphone  
- **Pose Classification** - Classify body poses and movements using pose estimation

All dashboards feature change-based MQTT publishing to minimize network traffic and provide responsive IoT integration.

## Features

- **Real-time Inference** - Live classification using webcam, microphone, or pose detection
- **MQTT Integration** - Publish classification results to any MQTT broker
- **Change-based Publishing** - Only publishes when detected class changes (reduces spam)
- **Configurable Thresholds** - Adjust confidence levels and sensitivity
- **Visual Feedback** - Live preview with confidence scores and pose skeleton overlay
- **Detailed Logging** - Debug information and connection status
- **Responsive UI** - Modern, mobile-friendly interface

## Quick Start

### 1. Train Your Model

1. Go to [Teachable Machine](https://teachablemachine.withgoogle.com/)
2. Choose your project type (Image, Audio, or Pose)
3. Collect samples for each class you want to detect
4. Train your model
5. Export your model and copy the shareable link

### 2. Set Up the Dashboard

1. Download the HTML file for your model type:
   - `image-classification.html` - For image models
   - `audio-classification.html` - For audio models  
   - `pose-classification.html` - For pose models

2. Open the HTML file in a modern web browser (Chrome, Firefox, Safari, Edge)

### 3. Configure the Dashboard

1. **MQTT Settings**:
   - **Broker URL**: Use `wss://broker.hivemq.com:8884/mqtt` for testing or your own MQTT broker
   - **Topic**: Set a unique topic like `myproject/classification`
   - Click "Connect to MQTT"

2. **Model Settings**:
   - **Model URL**: Paste your Teachable Machine shareable link
   - **Probability Threshold**: Adjust sensitivity (0.5 = more sensitive, 0.9 = very strict)
   - Click "Load Model"

3. **Start Classification**:
   - Click "Start Classification" to begin real-time inference
   - Grant camera/microphone permissions when prompted

## Hardware Integration

### Myduino AIoT Education Kit Setup

The classification results can control LEDs, LCD displays, servos, and other components on the Myduino AIoT Education Kit.

#### Arduino Code Example

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <LiquidCrystal_I2C.h>

// WiFi and MQTT settings
const char* ssid = "your-wifi-ssid";
const char* password = "your-wifi-password";
const char* mqtt_server = "broker.hivemq.com";
const char* mqtt_topic = "teachable-machine/classification";

// Hardware setup
LiquidCrystal_I2C lcd(0x27, 16, 2);
const int redLED = 2;
const int greenLED = 4;
const int blueLED = 5;

WiFiClient espClient;
PubSubClient client(espClient);

void setup_wifi(){
    Serial.print("Connecting to WiFi ")
    WiFi.begin(ssid, password);
    while(WiFi.status() != WL_CONNECTED){
        Serial.print(".");
        delay(500);
    }
    Serial.println("WiFi connected!");
}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Connecting to MQTT broker ...");
    // Attempt to connect
    if (client.connect("arduinoClient")) {
      Serial.println("connected");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  
  // Initialize hardware
  pinMode(redLED, OUTPUT);
  pinMode(greenLED, OUTPUT);  
  pinMode(blueLED, OUTPUT);
  lcd.init();
  lcd.backlight();
  
  // Connect to WiFi and MQTT
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void callback(char* topic, byte* payload, unsigned int length) {
  // Parse JSON message
  DynamicJsonDocument doc(1024);
  deserializeJson(doc, payload, length);
  
  String className = doc["className"];
  float probability = doc["probability"];
  
  // Update LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(className);
  lcd.setCursor(0, 1);
  lcd.print("Conf: " + String(probability * 100, 1) + "%");
  
  // Control LEDs based on classification
  digitalWrite(redLED, LOW);
  digitalWrite(greenLED, LOW);
  digitalWrite(blueLED, LOW);
  
  if (className == "Class1") {
    digitalWrite(redLED, HIGH);
  } else if (className == "Class2") {
    digitalWrite(greenLED, HIGH);
  } else if (className == "Class3") {
    digitalWrite(blueLED, HIGH);
  }
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();
}
```

## MQTT Message Format

All dashboards publish JSON messages in this format:

```json
{
  "type": "image",
  "className": "detected_class_name", 
  "probability": 0.95
}
```

- `type`: Classification type ("image", "audio", "pose")
- `className`: The detected class name from your model
- `probability`: Confidence score (0.0 to 1.0)

## Configuration Options

### Image Classification
- **Probability Threshold**: Minimum confidence to trigger MQTT publish
- **Model URL**: Your Teachable Machine image model link

### Audio Classification  
- **Probability Threshold**: Minimum confidence to trigger MQTT publish
- **Overlap Factor**: How much audio windows overlap (0.25 to 0.75)
- **Model URL**: Your Teachable Machine audio model link

### Pose Classification
- **Probability Threshold**: Minimum confidence to trigger MQTT publish  
- **Pose Confidence**: Minimum confidence for pose keypoints (0.3 to 0.9)
- **Model URL**: Your Teachable Machine pose model link

## Browser Compatibility

- **Chrome/Chromium**: Full support
- **Firefox**: Full support
- **Safari**: Full support (macOS 12+)
- **Edge**: Full support

**Note**: HTTPS is required for camera/microphone access. Use a local server or GitHub Pages for deployment.

## Troubleshooting

### Model Won't Load
- Verify the Teachable Machine URL ends with `/`
- Check browser console for CORS errors
- Ensure stable internet connection

### MQTT Connection Fails
- Verify broker URL format (must include `wss://` for secure websockets)
- Check if broker supports websocket connections
- Try the default HiveMQ broker for testing

### Camera/Microphone Access Denied
- Enable camera/microphone permissions in browser settings
- Use HTTPS (required for media access)
- Check if another application is using the camera

### No MQTT Messages
- Verify classification is above probability threshold
- Check MQTT connection status in logs
- Ensure the detected class is actually changing

## Advanced Usage

### Custom MQTT Brokers
Replace the default broker with your own:
- **AWS IoT**: `wss://your-endpoint.iot.region.amazonaws.com:443/mqtt`
- **Azure IoT**: Configure with connection strings
- **Local Mosquitto**: `wss://your-server:9001/mqtt`

### Multiple Device Control
Use different MQTT topics for different devices:
- `home/livingroom/lights`
- `factory/conveyor/control`
- `classroom/display/content`

### Integration Examples
- **Smart Home**: Control lights based on gestures
- **Industrial**: Quality control with image classification  
- **Healthcare**: Monitor exercise poses
- **Education**: Interactive learning with voice commands

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with different browsers and models
5. Submit a pull request

## License

MIT License - Feel free to use in personal and commercial projects.

## Acknowledgments

- [Google Teachable Machine](https://teachablemachine.withgoogle.com/) - For the amazing ML training platform
- [TensorFlow.js](https://www.tensorflow.org/js) - For browser-based ML inference
- [MQTT.js](https://github.com/mqttjs/MQTT.js) - For MQTT connectivity
- [Tailwind CSS](https://tailwindcss.com/) - For responsive styling

---

**Need help?** Open an issue or check the troubleshooting section above.