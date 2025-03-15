# How to Send Button Inputs to Nintendo Switch

## Overview

This document explains how to properly send button inputs to a Nintendo Switch using the sys-botbase module. This includes regular buttons, stick clicks (L3/R3), and analog stick positions.

## Critical Implementation Requirements

1. **Controller Configuration**
   ```python
   # MUST configure controller before sending any button commands exactly like this
   switch.sendCommand("configure controllerType 3")       # Set default controller type
   switch.sendCommand("configure buttonClickSleepTime 50") # Set button timing
   switch.sendCommand("configure mainLoopSleepTime 50")   # Set loop timing
   ```

2. **Command Formatting**
   - Every command MUST end with `\r\n`
   - Add 50ms sleep between commands
   ```python
   def sendCommand(self, content):
       content += '\r\n'  # Critical: Add \r\n at end
       self.socket.sendall(content.encode())
       time.sleep(0.05)  # Wait 50ms between commands
   ```

3. **Button Press/Release Protocol**
   ```python
   # Simple and reliable button implementation
   def press_button(self, button):
       self.sendCommand(f"press {button}")
       
   def release_button(self, button):
       self.sendCommand(f"release {button}")
   ```

4. **Connection Setup**
   ```python
   # Proper connection sequence
   socket.connect((ip_address, port))
   configure_controller()  # Send configuration commands
   sendCommand("getTitleID")  # Test connection
   ```

## Button Types

### Standard Buttons
The following buttons can be used with `press`, `release`, and `click` commands:

```
A, B, X, Y          # Face buttons
DUP, DDOWN,         # D-Pad
DLEFT, DRIGHT       
L, R                # Shoulder buttons
ZL, ZR              # Trigger buttons
PLUS, MINUS         # Plus/Minus buttons
LSTICK, RSTICK      # L3/R3 (stick clicks)
HOME, CAPTURE       # System buttons
```

### Example Usage

```python
# Click a button (press and release)
switch.click_button("A")

# Press and hold a button
switch.press_button("B")

# Release a held button
switch.release_button("B")

# Combine buttons in a sequence
switch.click_sequence("A,W1000,B")  # Press A, wait 1 second, press B
```

## Analog Stick Control

### Command Format
```python
# Format: setStick STICK_NAME X_VALUE Y_VALUE
# STICK_NAME must be either "LEFT" or "RIGHT" (case sensitive)
# X_VALUE and Y_VALUE are hexadecimal coordinates (-0x8000 to 0x7FFF)

# Examples:
switch.sendCommand("setStick LEFT 0x0000 0x7FFF")   # Move left stick up
switch.sendCommand("setStick RIGHT -0x8000 0x0000") # Move right stick left
```

### Quick Reference Values
```
STICK VALUES QUICK REFERENCE
---------------------------
Neutral:    X: 0x0000    Y: 0x0000    # Center position
Full Up:    X: 0x0000    Y: 0x7FFF    # Maximum up
Full Down:  X: 0x0000    Y: -0x8000   # Maximum down
Full Right: X: 0x7FFF    Y: 0x0000    # Maximum right
Full Left:  X: -0x8000   Y: 0x0000    # Maximum left

Diagonals (75% power for smooth movement):
Up-Right:   X: 0x5FFF    Y: 0x5FFF    # 75% up-right
Up-Left:    X: -0x5FFF   Y: 0x5FFF    # 75% up-left
Down-Right: X: 0x5FFF    Y: -0x5FFF   # 75% down-right
Down-Left:  X: -0x5FFF   Y: -0x5FFF   # 75% down-left

Partial Movement:
Half Right: X: 0x4000    Y: 0x0000    # 50% right (good for camera)
Half Up:    X: 0x0000    Y: 0x4000    # 50% up
Half Left:  X: -0x4000   Y: 0x0000    # 50% left
Half Down:  X: 0x0000    Y: -0x4000   # 50% down
```

### Stick Position Format
Analog sticks use a hexadecimal coordinate system:
- X-axis: -0x8000 (left) to 0x7FFF (right)
- Y-axis: -0x8000 (down) to 0x7FFF (up)  [IMPORTANT: Y is inverted!]
- Neutral: (0x0, 0x0)

### Verified Stick Positions

```python
# Left Stick Positions
LEFT_STICK = {
    "UP":    {"cmd": "setStick LEFT 0x0000 0x7FFF"},   # Full up
    "DOWN":  {"cmd": "setStick LEFT 0x0000 -0x8000"},  # Full down
    "RIGHT": {"cmd": "setStick LEFT 0x7FFF 0x0000"},   # Full right
    "LEFT":  {"cmd": "setStick LEFT -0x8000 0x0000"},  # Full left
    "CENTER": {"cmd": "setStick LEFT 0x0000 0x0000"}   # Neutral position
}

# Right Stick Positions
RIGHT_STICK = {
    "UP":    {"cmd": "setStick RIGHT 0x0000 0x7FFF"},   # Full up
    "DOWN":  {"cmd": "setStick RIGHT 0x0000 -0x8000"},  # Full down
    "RIGHT": {"cmd": "setStick RIGHT 0x7FFF 0x0000"},   # Full right
    "LEFT":  {"cmd": "setStick RIGHT -0x8000 0x0000"},  # Full left
    "CENTER": {"cmd": "setStick RIGHT 0x0000 0x0000"}   # Neutral position
}

# Diagonal Positions (75% tilt for smooth movement)
DIAGONAL = 0x5FFF  # ~75% of maximum value

# Left Stick Diagonals
LEFT_DIAGONALS = {
    "UP_RIGHT":   {"cmd": "setStick LEFT 0x5FFF 0x5FFF"},    # Up-right
    "UP_LEFT":    {"cmd": "setStick LEFT -0x5FFF 0x5FFF"},   # Up-left
    "DOWN_RIGHT": {"cmd": "setStick LEFT 0x5FFF -0x5FFF"},   # Down-right
    "DOWN_LEFT":  {"cmd": "setStick LEFT -0x5FFF -0x5FFF"}   # Down-left
}

# Right Stick Diagonals
RIGHT_DIAGONALS = {
    "UP_RIGHT":   {"cmd": "setStick RIGHT 0x5FFF 0x5FFF"},    # Up-right
    "UP_LEFT":    {"cmd": "setStick RIGHT -0x5FFF 0x5FFF"},   # Up-left
    "DOWN_RIGHT": {"cmd": "setStick RIGHT 0x5FFF -0x5FFF"},   # Down-right
    "DOWN_LEFT":  {"cmd": "setStick RIGHT -0x5FFF -0x5FFF"}   # Down-left
}
```

### Example Usage

```python
# Basic stick movement
def move_left_stick_up():
    switch.sendCommand("setStick LEFT 0x0000 0x7FFF")  # Move up
    time.sleep(0.5)  # Hold for half second
    switch.sendCommand("setStick LEFT 0x0000 0x0000")  # Return to center

def move_right_stick_diagonal():
    switch.sendCommand("setStick RIGHT 0x5FFF 0x5FFF")  # Move up-right
    time.sleep(0.5)  # Hold for half second
    switch.sendCommand("setStick RIGHT 0x0000 0x0000")  # Return to center

# Camera control example
def rotate_camera_right():
    switch.sendCommand("setStick RIGHT 0x4000 0x0000")  # 50% right
    time.sleep(0.2)  # Short duration for smooth camera
    switch.sendCommand("setStick RIGHT 0x0000 0x0000")  # Center camera

# Walking/running example
def walk_diagonal():
    switch.sendCommand("setStick LEFT -0x5FFF 0x5FFF")  # Up-left at 75%
    time.sleep(1.0)  # Walk for 1 second
    switch.sendCommand("setStick LEFT 0x0000 0x0000")  # Stop walking
```

### Critical Implementation Notes

1. **Stick Selection**
   - ALWAYS use "LEFT" or "RIGHT" exactly (case sensitive)
   - Invalid stick names will be ignored by the Switch

2. **Command Format**
   - Format must be: `setStick STICK_NAME X_VALUE Y_VALUE\r\n`
   - Values must be in hexadecimal format
   - Each command must end with `\r\n`

3. **Return to Neutral**
   - ALWAYS return sticks to neutral (0x0, 0x0) after movement
   - Failing to do so will leave the stick "stuck" in position
   - Example:
     ```python
     try:
         switch.sendCommand("setStick LEFT 0x7FFF 0x0000")  # Move right
         time.sleep(0.5)  # Hold for duration
     finally:
         switch.sendCommand("setStick LEFT 0x0000 0x0000")  # Always return to neutral
     ```

4. **Smooth Movement**
   - Use 75% values (0x5FFF) for diagonal movement
   - For camera control, use 50% values (0x4000) for precision
   - Full power (0x7FFF) can be too sensitive for some games

5. **Error Prevention**
   ```python
   def safe_stick_movement(stick, x, y, duration):
       try:
           switch.sendCommand(f"setStick {stick} {x} {y}")
           time.sleep(duration)
       finally:
           switch.sendCommand(f"setStick {stick} 0x0000 0x0000")
   ```

## Advanced Button Sequences

### Click Sequence Format
The `clickSeq` command accepts a comma-separated string with special formatting:

```python
# Format options:
# buttonType - Regular click
# +buttonType - Press and hold
# -buttonType - Release
# Wnumber - Wait for number milliseconds
# %X,Y - Move left stick to position X,Y
# &X,Y - Move right stick to position X,Y

# Examples:
# Hold ZL, press B, wait, release ZL
sequence = "+ZL,B,W1000,-ZL"

# Complex sequence with stick movement
sequence = "B,W500,%7FFF,0,W1000,%0,0,A"  # Press B, wait, full right, wait, center, press A
```

## Common Use Cases

### 1. Cardinal Direction Examples
```python
# Up Movement
switch.set_stick("LEFT", 0x0000, 0x8000)   # Full up (positive Y)
switch.wait(500)                           # Hold for half second
switch.set_stick("LEFT", 0x0, 0x0)         # Return to neutral

# Down Movement
switch.set_stick("LEFT", 0x0000, -0x8000)  # Full down (negative Y)
switch.wait(500)                           # Hold for half second
switch.set_stick("LEFT", 0x0, 0x0)         # Return to neutral

# Right Movement
switch.set_stick("LEFT", 0x7FFF, 0x0000)   # Full right (positive X)
switch.wait(500)                           # Hold for half second
switch.set_stick("LEFT", 0x0, 0x0)         # Return to neutral

# Left Movement
switch.set_stick("LEFT", -0x8000, 0x0000)  # Full left (negative X)
switch.wait(500)                           # Hold for half second
switch.set_stick("LEFT", 0x0, 0x0)         # Return to neutral

# CRITICAL: Remember Y-axis is inverted!
# Up    = positive Y (0x8000)
# Down  = negative Y (-0x8000)
# Right = positive X (0x7FFF)
# Left  = negative X (-0x8000)
```

### 2. Basic Movement
```python
# Walk right
switch.set_stick("LEFT", 0x4000, 0x0)  # 50% right
switch.wait(1000)                      # Wait 1 second
switch.set_stick("LEFT", 0x0, 0x0)     # Return to neutral

# Run right
switch.press_button("B")               # Hold B to run
switch.set_stick("LEFT", 0x7FFF, 0x0)  # Full right
switch.wait(1000)                      # Wait 1 second
switch.set_stick("LEFT", 0x0, 0x0)     # Return to neutral
switch.release_button("B")             # Release B
```

### 3. Camera Control
```python
# Look around
switch.set_stick("RIGHT", 0x4000, 0x0)  # Rotate camera right
switch.wait(500)
switch.set_stick("RIGHT", 0x0, 0x0)     # Center camera

# Click R3 to reset camera
switch.click_button("RSTICK")
```

### 4. Menu Navigation
```python
# Navigate menu with D-Pad
sequence = "DUP,W100,DRIGHT,W100,A"    # Up, right, select
switch.click_sequence(sequence)
```

## Best Practices

1. **Always Return Sticks to Neutral**
   - After moving sticks, return them to (0x0, 0x0)
   - Prevents unintended movement

2. **Use Appropriate Delays**
   - Add waits between inputs (W100-W500 ms)
   - Allows game to process inputs properly

3. **Handle Button States**
   - Track held buttons
   - Release held buttons before disconnecting

4. **Stick Movement**
   - Use moderate values for precise movement
   - Full stick values (0x7FFF) can be too sensitive

## Error Handling

```python
try:
    switch.click_button("A")
except ConnectionError:
    print("Lost connection to Switch")
except Exception as e:
    print(f"Error sending button input: {str(e)}")
```

## References
- sys-botbase Documentation: https://github.com/olliz0r/sys-botbase
- Button Types: See commands.md for full list
- Stick Coordinate System: 16-bit signed hex (-32768 to 32767) 