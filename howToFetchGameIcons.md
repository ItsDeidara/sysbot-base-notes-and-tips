# How to Fetch Game Icons from Nintendo Switch

## Overview

This document explains how to properly fetch and display game icons from a Nintendo Switch using the sys-botbase module. Game icons are sent in a special format that requires specific handling to display correctly.

## The Challenge

When fetching game icons from the Nintendo Switch using sys-botbase's `game icon` command, the data is returned as **hex-encoded text** rather than raw binary data. This causes issues when trying to display the image directly, as most image handling code expects binary data.

## Key Metrics

- Expected hex-encoded data size: ~180,000 characters
- Resulting binary data size: ~90,000 bytes
- JPEG header (hex): FFD8
- JPEG end marker (hex): FFD9
- Default socket timeout: 10 seconds
- Recommended buffer size: 4096 bytes

## Solution

The solution involves properly handling both hex-encoded and raw binary responses, with careful attention to buffer management and timeouts.

### 1. Backend (Python) Implementation

#### Receiving Binary Data with Hex-Encoding Support

```python
def _receive_binary_response(self, timeout=10):
    """Receive binary data from the Switch with hex-encoding handling"""
    if not self.socket:
        raise ConnectionError("Not connected to Switch")
        
    try:
        # Store and set socket timeout
        original_timeout = self.socket.gettimeout()
        self.socket.settimeout(timeout)
        
        try:
            # Read initial response
            initial_chunk = self.socket.recv(BUFFER_SIZE)
            
            if not initial_chunk:
                logger.warning("No data received from Switch")
                return None
            
            # Check for hex-encoded data
            try:
                text_data = initial_chunk.decode('utf-8', errors='ignore').strip()
                
                if text_data.startswith(JPEG_HEADER_HEX):
                    logger.info("Detected hex-encoded data")
                    full_text_data = text_data
                    start_time = time.time()
                    
                    # Receive until JPEG end marker
                    while True:
                        if time.time() - start_time > timeout:
                            logger.warning("Timeout while receiving hex data")
                            break
                            
                        if JPEG_END_HEX in full_text_data[-10:]:
                            logger.info("Received complete hex-encoded JPEG")
                            break
                            
                        try:
                            chunk = self.socket.recv(BUFFER_SIZE)
                            if not chunk:
                                break
                            full_text_data += chunk.decode('utf-8', errors='ignore')
                        except socket.timeout:
                            break
                    
                    logger.info(f"Received {len(full_text_data)} characters of hex data")
                    return full_text_data.encode('utf-8')
            except UnicodeDecodeError:
                pass
            
            # Handle raw binary data
            data = bytearray(initial_chunk)
            
            # Continue receiving until JPEG end marker
            while True:
                if len(data) >= 2 and data[-2:] == b'\xff\xd9':
                    break
                    
                try:
                    chunk = self.socket.recv(BUFFER_SIZE)
                    if not chunk:
                        break
                    data.extend(chunk)
                except socket.timeout:
                    break
            
            return bytes(data)
            
        finally:
            # Always restore original timeout
            self.socket.settimeout(original_timeout)
            
    except Exception as e:
        logger.error(f"Error receiving binary data: {str(e)}")
        return None
```

#### Fetching and Converting Game Icons

```python
@app.route('/api/switch/game-icon', methods=['GET'])
def get_game_icon():
    """Get the icon of the currently running game."""
    if not switch_controller.is_connected():
        return jsonify({'success': False, 'error': 'Not connected'})
    
    try:
        logger.info("Fetching game icon...")
        switch_controller._send_command("game icon")
        icon_data = switch_controller._receive_binary_response()
        
        if icon_data:
            try:
                # Check if we received hex-encoded data
                hex_str = icon_data.decode('utf-8', errors='ignore').strip()
                
                if hex_str.startswith(JPEG_HEADER_HEX):
                    logger.info("Received hex-encoded JPEG data, converting to binary")
                    binary_data = binascii.unhexlify(hex_str)
                    
                    if binary_data.startswith(b'\xff\xd8'):
                        logger.info(f"Successfully converted hex to binary: {len(binary_data)} bytes")
                        response = Response(binary_data, mimetype='image/jpeg')
                        response.headers['Cache-Control'] = 'no-store, no-cache, must-revalidate'
                        return response
            except:
                # If decoding/conversion fails, try using data as-is
                if icon_data.startswith(b'\xff\xd8'):
                    return Response(icon_data, mimetype='image/jpeg')
        
        logger.warning("No valid icon data received")
        return jsonify({'success': False, 'error': 'No valid icon data'})
        
    except Exception as e:
        logger.error(f"Error fetching game icon: {str(e)}")
        return jsonify({'success': False, 'error': str(e)})
```

## Important Notes

1. **Never Use `pixelPeek`**
   - Always use the `game icon` command
   - `pixelPeek` is for screen captures, not game icons
   - Using `pixelPeek` can cause timeouts and incorrect data

2. **Socket Timeout Management**
   - Store original timeout before changing
   - Use appropriate timeout for large data (10s recommended)
   - Always restore original timeout after receiving
   - Handle timeout exceptions gracefully

3. **Buffer Management**
   - Use consistent buffer size (4096 bytes recommended)
   - Check for JPEG end marker in last few bytes
   - Handle both hex-encoded and raw binary data
   - Concatenate buffers efficiently

4. **Data Validation**
   - Verify JPEG headers in both hex and binary form
   - Check data size (~180K chars hex, ~90K bytes binary)
   - Log data sizes for debugging
   - Include error context in logs

5. **Cache Prevention**
   - Set appropriate cache control headers
   - Add timestamp to image URLs
   - Handle browser caching properly

## Common Issues

1. **Timeouts**
   - Caused by incorrect socket timeout handling
   - Fixed by proper timeout management
   - Log timeout occurrences for debugging

2. **Incomplete Data**
   - Check for JPEG end marker
   - Implement proper buffer concatenation
   - Log data sizes for verification

3. **Invalid Image Data**
   - Verify JPEG headers
   - Handle both hex and binary formats
   - Provide clear error messages

## References

- JPEG File Format: https://www.fileformat.info/format/jpeg/
- sys-botbase Documentation: https://github.com/olliz0r/sys-botbase
- Socket Programming: https://docs.python.org/3/library/socket.html 