# ATTINY85 Cheat Sheet

## Sleep & Wake Example - Most MCU can "run" for years on coin cells. Just make better use of sleep and STOP states.

```cpp
#include <avr/sleep.h>
#include <avr/interrupt.h>

#define INTERRUPT_PIN 2  // The pin connected to the external interrupt

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(INTERRUPT_PIN, INPUT);  // Set the interrupt pin as input with a pull-up resistor
  attachInterrupt(digitalPinToInterrupt(INTERRUPT_PIN), wakeUp, RISING);  // Trigger on rising edge, connect pulldown resistor.
}

void loop() {
  // Go to sleep until interrupt is triggered
  goToSleep();

  // Wakes up here when interrupt happens
  // Do something after wake-up (e.g., blink LED, or handle event)
  // Example: Toggle an LED on wakeup (if you have an LED connected)
  digitalWrite(LED_BUILTIN, HIGH);
  delay(50);
  digitalWrite(LED_BUILTIN, LOW);
}

void goToSleep() {
  set_sleep_mode(SLEEP_MODE_PWR_DOWN);  // Set the sleep mode to power down
  sleep_enable();                       // Enable sleep mode

  // Disable ADC to save power
  ADCSRA &= ~(1 << ADEN);

  // Enable interrupts before going to sleep
  sei();
  sleep_cpu();  // Go to sleep

  // Program resumes here after waking up
  sleep_disable();
}

void wakeUp() {
  // This is the interrupt service routine (ISR)
  // The wake-up process is handled here, but nothing needs to be done
}
```

## Momentary Switch On/Off Trigger Example

```cpp
// ATtiny85 (Digispark). Arduino core: Digistump
// PWM out on P0 (PB0 / OC0A) -> 1k -> FemtoBuck DIM/CTRL
// Button on P2 to GND, using INPUT_PULLUP

const uint8_t PIN_PWM   = 0; // P0
const uint8_t PIN_BTN   = 2; // P2

bool isOn = false;

// crude but effective debounce
bool readButtonDebounced() {
  static uint32_t tLast = 0;
  static uint8_t  last  = HIGH;
  uint8_t now = digitalRead(PIN_BTN);
  if (now != last) { tLast = millis(); last = now; }
  return (millis() - tLast) > 30 ? now : HIGH; // stable LOW when pressed
}

void setup() {
  pinMode(PIN_BTN, INPUT_PULLUP);
  pinMode(PIN_PWM, OUTPUT);

  // Set Timer0 PWM to a higher frequency (~1 kHz instead of ~490 Hz)
  // Optional: default is fine for the AL8805, but this keeps flicker out of sight.
  // TCCR0B = (TCCR0B & 0b11111000) | 0x02; // prescaler /8 -> ~1kHz on OC0A/OC0B

  // Start OFF
  analogWrite(PIN_PWM, 0); // 0% duty -> DIM < 0.4 V -> driver OFF
}

void loop() {
  static bool prev = HIGH;
  bool b = readButtonDebounced();

  // detect press (ACTIVE LOW)
  if (prev == HIGH && b == LOW) {
    isOn = !isOn;
    analogWrite(PIN_PWM, isOn ? 255 : 0); // 100% or 0% duty
  }
  prev = b;
}
```

## Momentary Switch On/Off Trigger Example, With Debounce & States

```cpp
// ATtiny85 (Digispark-style, 16.5 MHz core)
// P0 (PB0/OC0A) -> 1k -> FemtoBuck DIM/CTRL
// P2 (PB2) -> button to GND (INPUT_PULLUP enabled)

const uint8_t PIN_PWM = 0;  // P0: OC0A
const uint8_t PIN_BTN = 2;  // P2: button (to GND)

// ---- Timing & behaviour constants ----
const uint16_t DEBOUNCE_MS       = 30;    // debounce for button edges
const uint16_t HOLD_THRESHOLD_MS = 350;   // press longer than this counts as "hold"
const uint16_t ADJUST_WINDOW_MS  = 2000;  // after turning on, how long quick presses dim in steps
const uint16_t RAMP_TIME_MS      = 3000;  // 0% -> 100% ramp duration when holding

// Step levels (100%, 75%, 50%, 25%, 0%)
const uint8_t STEP_PWM[] = {255, 191, 128, 64, 0};
const uint8_t NUM_STEPS  = sizeof(STEP_PWM);

// ---- State machine ----
enum State : uint8_t {
  OFF_STATE = 0,
  ON_ADJUST,   // within adjust window: quick presses step 100->75->50->25->0
  ON_LOCKED,   // after adjust window: next press turns OFF
  RAMPING      // holding from OFF ramps 0->100 over 3s
};

State state = OFF_STATE;

// ---- Book-keeping ----
uint8_t  stepIndex      = 0;      // index into STEP_PWM when in ON_ADJUST
uint32_t lastStableTime = 0;      // for debounce
bool     btnStable      = true;   // stable debounced level
bool     btnPrevStable  = true;

uint32_t pressStartMs   = 0;      // time when a press started (debounced)
uint32_t adjustDeadline = 0;      // time when ON_ADJUST -> ON_LOCKED

uint32_t rampStartMs    = 0;      // when ramp began

// ---- Helpers ----
// CHANGED: track current PWM so we can lock to it after a hold
volatile uint8_t pwmLevel = 0;    // last value written to PWM (0..255)

void setPWM(uint8_t val) {
  pwmLevel = val;                 // CHANGED
  analogWrite(PIN_PWM, val);
}

uint8_t currentPWM() {
  return pwmLevel;                // CHANGED
}

// Debounced button read (returns HIGH when not pressed, LOW when pressed)
bool readButtonDebounced() {
  bool raw = digitalRead(PIN_BTN);
  if (raw != btnStable) {
    if (millis() - lastStableTime >= DEBOUNCE_MS) {
      btnStable = raw;
      lastStableTime = millis();
    }
  } else {
    lastStableTime = millis();
  }
  return btnStable;
}

void enterOff() {
  state = OFF_STATE;
  setPWM(0);
}

void enterOnAdjustAt100() {
  state = ON_ADJUST;
  stepIndex = 0;              // 100%
  setPWM(STEP_PWM[stepIndex]);
  adjustDeadline = millis() + ADJUST_WINDOW_MS;
}

void stepDimOrOff() {
  if (stepIndex + 1 < NUM_STEPS) {
    stepIndex++;
    setPWM(STEP_PWM[stepIndex]);
    if (STEP_PWM[stepIndex] == 0) {
      enterOff();
    } else {
      // Extend the window so rapid multi-press stepping stays responsive
      adjustDeadline = millis() + ADJUST_WINDOW_MS;
    }
  } else {
    // Already at 0 -> ensure OFF
    enterOff();
  }
}

void enterOnLocked(uint8_t pwmVal) {
  state = ON_LOCKED;
  setPWM(pwmVal);
}

void startRamp() {
  state = RAMPING;
  rampStartMs = millis();
  setPWM(0); // start from 0%
}

void updateRamp() {
  uint32_t elapsed = millis() - rampStartMs;
  if (elapsed >= RAMP_TIME_MS) {
    setPWM(255); // cap at 100%
    // Stay in RAMPING until release; no further increase
  } else {
    // Linear ramp 0..255 over RAMP_TIME_MS
    uint16_t pwm = (uint32_t)255 * elapsed / RAMP_TIME_MS;
    setPWM((uint8_t)pwm);
  }
}

void setup() {
  pinMode(PIN_PWM, OUTPUT);
  pinMode(PIN_BTN, INPUT_PULLUP);

  // Optional: tweak Timer0 PWM freq (uncomment if you want ~1 kHz)
  // TCCR0B = (TCCR0B & 0b11111000) | 0x02;

  enterOff();
  btnStable = digitalRead(PIN_BTN);
  btnPrevStable = btnStable;
  lastStableTime = millis();
}

void loop() {
  bool btn = readButtonDebounced();  // HIGH = released, LOW = pressed

  // Detect edges
  bool fell  = (btnPrevStable == HIGH && btn == LOW);
  bool rose  = (btnPrevStable == LOW  && btn == HIGH);

  // State-independent: handle timing transitions
  if (state == ON_ADJUST && (int32_t)(millis() - adjustDeadline) >= 0) {
    // Adjust window expired -> lock level
    enterOnLocked(STEP_PWM[stepIndex]);
  }

  // Ramping updates while held
  if (state == RAMPING && btn == LOW) {
    updateRamp();
  }

  // Edge-driven logic
  if (fell) {
    // Button pressed
    pressStartMs = millis();

    if (state == OFF_STATE) {
      // We won't act immediately—wait to see if it's a hold.
      // If it becomes a hold, we'll startRamp() after HOLD_THRESHOLD_MS in the loop below.
    }
  }

  // Long-press detection from OFF (begin ramp once threshold exceeded and still held)
  if (state == OFF_STATE && btn == LOW) {
    if ((millis() - pressStartMs) >= HOLD_THRESHOLD_MS) {
      startRamp();
    }
  }

  if (rose) {
    // Button released
    uint32_t pressDur = millis() - pressStartMs;

    switch (state) {
      case OFF_STATE:
        if (pressDur < HOLD_THRESHOLD_MS) {
          // Short press from OFF -> ON at 100% and start adjust window
          enterOnAdjustAt100();
        } else {
          // Rare race: lock whatever PWM we actually had (not 0)
          enterOnLocked(currentPWM());     // CHANGED
        }
        break;

      case RAMPING: {
        // Stop ramp at current brightness; next press should turn off
        enterOnLocked(currentPWM());       // CHANGED
        break;
      }

      case ON_ADJUST:
        if (pressDur < HOLD_THRESHOLD_MS) {
          // Within window: step down 100->75->50->25->0
          stepDimOrOff();
        } else {
          // Long hold while in adjust—no special action
        }
        break;

      case ON_LOCKED:
        if (pressDur < HOLD_THRESHOLD_MS) {
          // Locked behaviour: any short press turns OFF
          enterOff();
        } else {
          // Long hold while locked: keep as-is (no special behaviour defined)
        }
        break;
    }
  }

  btnPrevStable = btn;
}
```
