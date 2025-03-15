# How to Fetch Game Information from Nintendo Switch

## Overview

This document explains how to properly fetch game information from a Nintendo Switch using sys-botbase commands. There are four key pieces of information we can retrieve:

1. Title ID
2. Game Name
3. Game Version
4. Game Author

## Commands

### Getting Title ID
```python
# Use the getTitleID command
response = send_command("getTitleID")
# Returns: e.g., "01003D200BAA2000"
```

### Getting Game Info
```python
# Use the game command with specific parameters
name = send_command("game name")      # Returns: e.g., "Pokémon Mystery Dungeon Rescue Team DX"
version = send_command("game version") # Returns: e.g., "1.0.2"
author = send_command("game author")   # Returns: e.g., "Nintendo"
```

## Key Principles

1. **Check Title ID First**
   - Title ID of all zeros ("0000000000000000") means no game is running
   - Only try to fetch other info if you have a valid title ID
   - Title ID is the most reliable way to check if a game is running

2. **Keep It Simple**
   - Don't add extra validation or processing
   - Let the Switch handle the responses
   - Pass through responses directly to the client

3. **Handle Errors Gracefully**
   - Return meaningful defaults when no game is running
   - Don't try to parse error messages, just check if command succeeded
   - Use consistent fallback values (e.g., "-" for missing info)

## Example Implementation

```python
def get_game_info():
    """Get information about the currently running game"""
    # First get title ID
    title_id = get_title_id()
    
    # Only try to get game info if we have a non-zero title ID
    if title_id and title_id != "0000000000000000":
        info = {
            'title_id': title_id,
            'name': get_game_info('name'),
            'version': get_game_info('version'),
            'author': get_game_info('author')
        }
    else:
        info = {
            'title_id': '0000000000000000',
            'name': 'No game running',
            'version': '-',
            'author': '-'
        }
    
    return info
```

## Common Issues

1. **nsGetApplicationControlData() failed: 0x6410**
   - This usually means no game is running
   - Check title ID is not all zeros before trying other commands
   - Return default values instead of error messages

2. **Empty or Invalid Responses**
   - Some games might not provide all information
   - Always have fallback values ready
   - Use consistent defaults across your application

3. **Command Order**
   - Always get title ID first
   - Only fetch other info if title ID is valid
   - This prevents unnecessary commands when no game is running

## Best Practices

1. **Minimize Commands**
   - Only fetch what you need
   - Cache results if appropriate
   - Don't spam commands unnecessarily

2. **Consistent Defaults**
   - Use "-" for missing text information
   - Use "0000000000000000" for missing title ID
   - Use "No game running" for name when appropriate

3. **Error Handling**
   - Keep error handling minimal
   - Focus on the happy path
   - Let errors propagate naturally

## References

- sys-botbase documentation
- Nintendo Switch game metadata format
- Application Control Data structure 