# How to Fetch Game Icons from Nintendo Switch

## Overview

This document explains how to properly fetch and display game icons from a Nintendo Switch using the sys-botbase module. Game icons are sent in a special format that requires specific handling to display correctly.

## The Challenge

When fetching game icons from the Nintendo Switch using sys-botbase commands like `game icon` or `pixelPeek`, the data is returned as **hex-encoded text** rather than raw binary data. This causes issues when trying to display the image directly, as most image handling code expects binary data.

## Solution

The solution involves detecting when the response is hex-encoded and converting it to binary before serving it as an image. Here's how we implemented it:

### 1. Backend (Python) Implementation

#### Detecting and Converting Hex-Encoded Images

```python
@app.route('/api/switch/game-icon', methods=['GET'])
def get_game_icon():
    """Get the icon of the currently running game on the Switch."""
    import io
    import base64
    import binascii
    
    if not switch_controller.is_connected():
        return jsonify({'success': False, 'error': 'Not connected to the Switch'})
    
    try:
        # Get the game icon using the game icon command
        logger.info("Fetching game icon...")
        
        # First try using the game icon command
        switch_controller._send_command("game icon")
        icon_data = switch_controller._receive_binary_response()
        
        if not icon_data or len(icon_data) < 100:  # Too small to be a valid image
            logger.warning("Game icon data is empty or too small, trying pixelPeek...")
            # Fallback to pixelPeek command
            switch_controller._send_command("pixelPeek")
            icon_data = switch_controller._receive_binary_response()
        
        # Check if we received a hex-encoded string instead of binary data
        if icon_data and len(icon_data) > 10:
            try:
                # Try to decode as text to check if it's hex-encoded
                hex_str = icon_data.decode('utf-8', errors='ignore').strip()
                
                # Check if it starts with the JPEG header in hex (FFD8)
                if hex_str.startswith('FFD8'):
                    logger.info("Received hex-encoded JPEG data, converting to binary")
                    try:
                        # Convert hex string to binary
                        binary_data = binascii.unhexlify(hex_str)
                        
                        # Verify it's a valid JPEG
                        if binary_data.startswith(b'\xff\xd8'):
                            logger.info(f"Successfully converted hex to binary: {len(binary_data)} bytes")
                            
                            # Return the binary data as a JPEG image with cache control headers
                            response = Response(binary_data, mimetype='image/jpeg')
                            response.headers['Cache-Control'] = 'no-store, no-cache, must-revalidate, max-age=0'
                            response.headers['Pragma'] = 'no-cache'
                            response.headers['Expires'] = '0'
                            return response
                    except binascii.Error as e:
                        logger.warning(f"Error converting hex to binary: {str(e)}")
            except Exception as e:
                logger.warning(f"Error processing potential hex data: {str(e)}")
        
        # If we reach here, either we didn't get hex data or conversion failed
        # Handle as regular binary data...
```

#### Receiving Binary (or Hex) Data from the Switch

```python
def _receive_binary_response(self, buffer_size: int = 4096, timeout: int = 10) -> bytes:
    """
    Receive a binary response from the Switch (for images and other binary data).
    """
    # ... setup code ...
    
    try:
        # First, read the initial response
        initial_chunk = self.socket.recv(buffer_size)
        
        if not initial_chunk:
            logger.warning("No data received from Switch")
            return bytes()
        
        # Check if the response is a hex-encoded string (common for JPEG data from sys-botbase)
        try:
            # Try to decode as text to see if it's a hex string
            text_data = initial_chunk.decode('utf-8', errors='ignore').strip()
            
            # If it starts with FFD8 (JPEG header in hex), it's likely a hex-encoded JPEG
            if text_data.startswith('FFD8'):
                logger.info("Detected hex-encoded data")
                
                # Continue receiving data until we get a complete response
                full_text_data = text_data
                
                # Keep receiving until we find the JPEG end marker (FFD9) or timeout
                while True:
                    # ... timeout and completion checks ...
                    
                    # Receive more data and append to our text data
                    chunk = self.socket.recv(buffer_size)
                    full_text_data += chunk.decode('utf-8', errors='ignore')
                
                # Return the hex-encoded data as bytes
                logger.info(f"Received {len(full_text_data)} characters of hex data")
                return full_text_data.encode('utf-8')
        except UnicodeDecodeError:
            # Not text data, continue with binary processing
            pass
        
        # ... regular binary data handling ...
```

### 2. Frontend (JavaScript) Implementation

```javascript
// Load game icon if available
if (data.has_icon) {
    // Add a timestamp to prevent caching
    const timestamp = new Date().getTime();
    const iconUrl = `/api/switch/game-icon?t=${timestamp}`;
    
    addLogEntry('Attempting to load game icon...');
    
    // Show loading state
    gameIcon.classList.add('loading');
    
    // Create a new image element to test loading
    const testImage = new Image();
    
    testImage.onload = function() {
        // Image loaded successfully, update the actual image
        gameIcon.src = iconUrl;
        gameIcon.classList.remove('loading');
        addLogEntry('Game icon loaded successfully');
        console.log('Game icon loaded successfully');
    };
    
    testImage.onerror = function(e) {
        // If loading fails, show default icon
        console.error('Failed to load game icon:', e);
        addLogEntry('Failed to load game icon - using default');
        gameIcon.src = '/static/img/default-game-icon.png';
        gameIcon.classList.remove('loading');
    };
    
    // Start loading the test image
    testImage.src = iconUrl;
    
    // Set a timeout to handle cases where the image might hang
    setTimeout(() => {
        if (gameIcon.classList.contains('loading')) {
            addLogEntry('Game icon load timed out - using default');
            gameIcon.src = '/static/img/default-game-icon.png';
            gameIcon.classList.remove('loading');
        }
    }, 5000);
}
```

## Key Insights

1. **Hex-Encoded Data**: The Switch sends JPEG images as hex-encoded strings, not raw binary data.

2. **JPEG Headers**: 
   - The hex-encoded JPEG data starts with "FFD8" (the JPEG file signature in hex)
   - The binary JPEG data starts with bytes `b'\xff\xd8'`

3. **Conversion Process**:
   - Detect if the response is text and starts with "FFD8"
   - Use `binascii.unhexlify()` to convert the hex string to binary
   - Verify the converted data starts with the JPEG binary signature
   - Serve the binary data with the correct MIME type

4. **Cache Prevention**:
   - Add cache control headers on the server side
   - Add a timestamp parameter to the image URL on the client side

5. **Error Handling**:
   - Implement fallbacks (try `game icon` first, then `pixelPeek`)
   - Add timeouts to prevent hanging requests
   - Provide a default icon for when fetching fails

## Debugging Tips

1. Check the logs for messages like:
   - "Received text response instead of binary data: FFD8FFE0..."
   - This indicates you're receiving hex-encoded data

2. Verify the first few bytes of the received data:
   - For hex-encoded data, it should start with "FFD8"
   - For binary data, it should start with bytes `b'\xff\xd8'`

3. If conversion fails, check for incomplete data:
   - JPEG data should end with "FFD9" (hex) or `b'\xff\xd9'` (binary)

## Common Issues

1. **304 Status Codes**: Browser caching the default icon instead of fetching a new one
   - Solution: Add cache control headers and URL parameters

2. **Small Data Size**: Received data is too small to be a valid image
   - Solution: Check data size and try alternative commands

3. **Incomplete Data**: JPEG data is cut off before the end marker
   - Solution: Implement proper data collection until end marker is found

## References

- JPEG File Format: https://www.fileformat.info/format/jpeg/
- sys-botbase Documentation: https://github.com/olliz0r/sys-botbase 