# How to Send Button Inputs to Nintendo Switch

## Overview

This document explains how to properly send button inputs to a Nintendo Switch using the sys-botbase module. This includes regular buttons, stick clicks (L3/R3), and analog stick positions.

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

### Stick Position Format
Analog sticks use a hexadecimal coordinate system:
- X-axis: -0x8000 (left) to 0x7FFF (right)
- Y-axis: -0x8000 (down) to 0x7FFF (up)  [IMPORTANT: Y is inverted!]
- Neutral: (0x0, 0x0)

### Verified Stick Positions

```python
# Cardinal Directions (Full Tilt)
UP    = { "x": 0x0000, "y": 0x8000 }   # Full up (positive Y)
DOWN  = { "x": 0x0000, "y": -0x8000 }  # Full down (negative Y)
RIGHT = { "x": 0x7FFF, "y": 0x0000 }   # Full right (positive X)
LEFT  = { "x": -0x8000, "y": 0x0000 }  # Full left (negative X)

# Diagonal Directions (75% Tilt for Smooth Movement)
DIAGONAL = 0x5FFF  # ~75% of maximum value
UP_RIGHT    = { "x": 0x5FFF, "y": 0x5FFF }
UP_LEFT     = { "x": -0x5FFF, "y": 0x5FFF }
DOWN_RIGHT  = { "x": 0x5FFF, "y": -0x5FFF }
DOWN_LEFT   = { "x": -0x5FFF, "y": -0x5FFF }

# Example Usage:
switch.set_stick("LEFT", UP["x"], UP["y"])      # Move left stick up
switch.set_stick("RIGHT", DOWN["x"], DOWN["y"])  # Move right stick down
```

### Critical Implementation Notes

1. **Y-Axis Inversion**
   - The Y-axis is INVERTED from standard coordinate systems
   - Positive Y (0x8000) moves UP
   - Negative Y (-0x8000) moves DOWN
   - Getting this wrong will result in inverted controls

2. **Diagonal Movement**
   - Use 75% of maximum value (0x5FFF) for smooth diagonal movement
   - Full power diagonals (0x7FFF) can be too sensitive
   - This provides better control for precise movements

3. **Return to Neutral**
   - ALWAYS return sticks to neutral position (0x0, 0x0) after movement
   - Failing to do so will leave the stick "stuck" in position

4. **Value Ranges**
   - X-axis maximum is asymmetric: -32768 to 32767 (-0x8000 to 0x7FFF)
   - Y-axis maximum is asymmetric: -32768 to 32767 (-0x8000 to 0x7FFF)
   - Neutral position is always (0x0, 0x0)

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