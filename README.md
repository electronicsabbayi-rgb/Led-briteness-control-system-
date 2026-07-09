# Led-briteness-control-system-




#define TRIG_PIN 9
#define ECHO_PIN 10
#define LED_PIN 6   // PWM pin

// Tunables (adjust for your setup)
const int NEAR_CM = 5;     // closest hand distance to start mapping
const int FAR_CM  = 30;    // farthest hand distance to still control brightness
const int OFF_AFTER_MS = 1500; // turn LED off if no valid hand for this long

// Smoothing
float emaDistance = 0;          // exponential moving average
const float ALPHA = 0.25f;      // 0..1 (higher = faster response)

unsigned long lastValidGestureMs = 0;

long readDistanceCM() {
  // Trigger ultrasonic pulse
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  // Measure echo time (with timeout to avoid lockups)
  long duration = pulseIn(ECHO_PIN, HIGH, 30000UL); // 30ms ~ 5m
  if (duration == 0) return -1; // timeout/no reading

  long cm = duration * 0.034 / 2; // speed of sound conversion
  return cm;
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  analogWrite(LED_PIN, 0);
  Serial.begin(9600);
}

void loop() {
  long d = readDistanceCM();
  if (d > 0 && d < 200) { // sanity range
    if (emaDistance == 0) emaDistance = d;     // initialize EMA on first good read
    emaDistance = ALPHA * d + (1 - ALPHA) * emaDistance;
  }

  // Check if the smoothed distance is within gesture window
  if (emaDistance >= NEAR_CM && emaDistance <= FAR_CM) {
    // Map distance -> brightness (near = bright, far = dim)
    int brightness = map((int)emaDistance, NEAR_CM, FAR_CM, 255, 0);
    brightness = constrain(brightness, 0, 255);
    analogWrite(LED_PIN, brightness);
    lastValidGestureMs = millis();

    Serial.print("Distance(cm): ");
    Serial.print((int)emaDistance);
    Serial.print("  Brightness: ");
    Serial.println(brightness);
  } else {
    // If no valid gesture for a while, fade out to clean idle
    if (millis() - lastValidGestureMs > OFF_AFTER_MS) {
      analogWrite(LED_PIN, 0);
    }
  }

  delay(25); // small loop delay for stability
}