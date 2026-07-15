# IoT-Energy-Harvesting-Sidewalk

Piezo Current Monitor (ESP32 + LCD + OLED RoboEyes)

An ESP32 sketch that measures current from a piezo/rectifier circuit using an ADC shunt, displays the reading on a 16x2 I2C LCD, and drives an animated "RoboEyes" character on an SH1106 OLED as a button-controlled billboard.

Features


Auto-zeroing current sensing — measures current across a shunt resistor via two ADC pins, with automatic offset calibration at boot and periodic re-zeroing during idle periods to correct for drift.
16x2 LCD readout — shows live current in mA (or A above 1000 mA) once it exceeds a configurable threshold; otherwise shows "Waiting for input".
OLED RoboEyes billboard — animated eyes (via the FluxGarage RoboEyes library) that react only to button presses, not sensor activity:

1 press → Angry mood + random message
2 presses (within 2s) → Happy mood + random message
3+ presses → Tired mood + random message
Reverts to default idle animation 3 seconds after the last press



I2C bus scanner — prints all detected I2C device addresses at boot for easy debugging.


Hardware Requirements

ComponentNotesESP32 dev boardAny standard ESP3216x2 I2C LCDDefault address 0x27SH1106 128x64 OLEDI2C, address 0x3CPush buttonConnected to pin 4, uses internal pull-upShunt resistorDefault 10.0 Ω — must match your actual resistorPiezo/rectifier circuitConnected across the shunt

Wiring

SignalESP32 PinI2C SDA21I2C SCL22Button4 (to GND, internal pull-up enabled)Shunt high side35Shunt low side32

Required Libraries

Install these via the Arduino Library Manager (or PlatformIO):


Wire (built-in)
LiquidCrystal_I2C
Adafruit GFX Library
Adafruit SH110X
FluxGarage RoboEyes


Configuration

Key constants near the top of the sketch you may need to adjust:

cpp#define LCD_ADDRESS 0x27          // I2C address of your LCD
const float SHUNT_RESISTOR = 10.0;      // Actual shunt resistor value (ohms)
const float CURRENT_THRESHOLD_mA = 3.0; // Below this reads as "no input"


SHUNT_RESISTOR: Set this to the real measured value of your resistor for accurate current readings.
CURRENT_THRESHOLD_mA: Tune this against your actual noise floor. With the piezo disconnected, watch the Serial Monitor output and set the threshold safely above the observed noise.


How It Works


Setup: Scans the I2C bus, initializes the LCD and OLED, and runs a 100-sample auto-zero calibration on the ADC shunt (keep the piezo untouched during this step).
Loop:

Reads the shunt voltage differentially across PIN_HIGH/PIN_LOW, subtracts the calibrated offset, and converts to current.
Updates the LCD every 500 ms with the current reading or a "Waiting for input" message.
Polls the button (debounced) and cycles through Angry → Happy → Tired moods based on press count within a 2-second window.
Renders the RoboEyes animation and any temporary text overlay on the OLED every loop iteration.
Automatically re-zeros the ADC offset if the current has read zero for 15+ seconds and it's been at least 60 seconds since the last calibration.





Serial Monitor

Open at 115200 baud to see:


Detected I2C addresses at startup
The calibrated auto-zero offset voltage
Live current readings (mA) once per LCD update cycle


Notes / Known Behavior


The OLED does not react to piezo activity — it only changes mood/text on button presses, by design.
The LCD shows current only; no wattage, voltage, or energy accumulation.
Mood/text automatically resets to the default idle animation 3 seconds after the last button press.
