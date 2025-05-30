# CometExample

A template for creating plugins (Comets) for the Sunshine system using the Corona framework.

## 🌟 What is a Comet?

A Comet is a standalone executable plugin that:
- Runs as an independent process
- Communicates with other Comets via SolarFlares (messages)
- Automatically registers with SunshineCore
- Handles its own lifecycle gracefully
- Can be distributed as a single executable file

## 🚀 Quick Start

### For Developers

1. **Clone this template:**
   ```bash
   git clone <comet-example-repo>
   cd CometExample
   ```

2. **Test in development mode:**
   ```bash
   ./run_dev.sh
   ```
   This shows a console window for debugging.

3. **Build the executable:**
   ```bash
   ./build_prod.sh
   ```

### For Users

1. Download the pre-built Comet executable
2. Copy to `Documents/Sunshine/plugins/`
3. Start SunshineCore - your Comet will launch automatically

## 📁 Project Structure

```
CometExample/
├── run_dev.sh        # Development runner (shows console)
├── build_prod.sh     # Production build script
├── README.md         # This file
└── comet/            # Comet application
    ├── Pipfile       # Python dependencies
    └── src/
        ├── main.py   # YOUR CODE GOES HERE
        └── corona/   # Framework (don't modify)
            ├── CometCore.py     # Lifecycle management
            ├── SolarFlare.py    # Message format
            ├── Satellite.py     # ZeroMQ handler
            └── crash_handler.py # Error logging
```

## 💻 Writing Your Comet

### Basic Template

```python
#!/usr/bin/env python3
from queue import Queue
from datetime import datetime
from corona import CometCore, SolarFlare

# Message queues
in_queue = Queue()
out_queue = Queue()

def on_startup():
    """Called when your Comet starts."""
    print("🚀 Starting up!")

def on_shutdown():
    """Called when your Comet shuts down."""
    print("👋 Shutting down!")

def main_loop(is_running):
    """Your main logic goes here."""
    while is_running():  # IMPORTANT: Use is_running()!
        
        # Check for messages
        if not in_queue.empty():
            flare = in_queue.get()
            handle_message(flare)
        
        # Do your work
        do_something()
        
        # Send messages
        flare = SolarFlare(
            timestamp=datetime.now(),
            name="MyComet",
            type="STATUS",
            payload={"status": "working"}
        )
        out_queue.put(flare)
        
        time.sleep(1)

# Create and start
if __name__ == "__main__":
    comet = CometCore(
        name="MyComet",
        subscribe_to=["COMMAND", "DATA"],  # Message types to receive
        in_queue=in_queue,
        out_queue=out_queue,
        on_startup=on_startup,
        on_shutdown=on_shutdown,
        main_loop=main_loop
    )
    comet.start()
```

### Important: The is_running() Function

⚠️ **Always use `is_running()` in your main loop!**

```python
# ✅ CORRECT - Will shutdown properly
def main_loop(is_running):
    while is_running():
        # your code

# ❌ WRONG - Will not shutdown properly
def main_loop(is_running):
    while True:  # Don't do this!
        # your code
```

## 📨 Message System

### Sending Messages

```python
from corona import SolarFlare
from datetime import datetime

# Create a message
flare = SolarFlare(
    timestamp=datetime.now(),
    name="MyComet",           # Your Comet's name
    type="CUSTOM_MESSAGE",    # Any string you want
    payload={                 # Any JSON-serializable data
        "command": "process",
        "data": [1, 2, 3]
    }
)

# Send it
out_queue.put(flare)
```

### Receiving Messages

```python
def main_loop(is_running):
    while is_running():
        if not in_queue.empty():
            flare = in_queue.get()
            
            # Check message type
            if flare.type == "COMMAND":
                process_command(flare.payload)
            elif flare.type == "DATA_REQUEST":
                send_data_response(flare.name)
```

### Message Filtering

Specify which message types you want to receive:

```python
# Only receive specific types
subscribe_to=["COMMAND", "STATUS", "DATA"]

# Receive everything
subscribe_to=["*"]
```

System messages (PING, PONG, etc.) are handled automatically by Corona.

## 🛡️ Error Handling

All crashes are automatically logged to:
- **Windows**: `%USERPROFILE%\Documents\Sunshine\Crash\`
- **Mac/Linux**: `~/Documents/Sunshine/Crash/`

Log files include:
- Full exception traceback
- Timestamp
- Comet name
- System information

### Manual Error Logging

```python
from corona import log_crash

try:
    risky_operation()
except Exception as e:
    log_crash("MyComet", "Failed to process data", e)
```

## 🔧 Configuration

### Development vs Production

- **Development Mode** (`--dev` flag): Shows console window
- **Production Mode**: Runs silently in background

### Dependencies

Add Python packages to `comet/Pipfile`:

```toml
[packages]
requests = "*"
numpy = "*"
```

Then rebuild your Comet.

## 📦 Distribution

1. **Build your Comet:**
   ```bash
   ./build_prod.sh
   ```

2. **Find the executable:**
   - Windows: `dist/YourComet.exe`
   - Mac/Linux: `dist/YourComet`

3. **Distribute:**
   - Upload to GitHub releases
   - Share the single executable file
   - Users just drop it in their plugins folder

## 🎯 Best Practices

### DO:
- ✅ Use `is_running()` in your main loop
- ✅ Handle exceptions gracefully
- ✅ Send periodic status updates
- ✅ Use meaningful message types
- ✅ Clean up resources in `on_shutdown()`

### DON'T:
- ❌ Use `while True` loops
- ❌ Block the main loop for long operations
- ❌ Modify the corona framework files
- ❌ Assume messages arrive in order
- ❌ Store large data in message payloads

## 🌈 Example Comets

### Simple Logger Comet

```python
def main_loop(is_running):
    log_file = open("activity.log", "a")
    
    while is_running():
        if not in_queue.empty():
            flare = in_queue.get()
            log_file.write(f"{flare.timestamp}: {flare.type} from {flare.name}\n")
            log_file.flush()
        time.sleep(0.1)
    
    log_file.close()
```

### Data Processor Comet

```python
def main_loop(is_running):
    while is_running():
        if not in_queue.empty():
            flare = in_queue.get()
            
            if flare.type == "PROCESS_REQUEST":
                data = flare.payload["data"]
                result = process_data(data)
                
                response = SolarFlare(
                    timestamp=datetime.now(),
                    name="DataProcessor",
                    type="PROCESS_RESPONSE",
                    payload={"result": result, "request_id": flare.payload["id"]}
                )
                out_queue.put(response)
```

## 🆘 Troubleshooting

### Comet won't start
- Check crash logs in `Documents/Sunshine/Crash/`
- Ensure SunshineCore is running first
- Verify ZeroMQ ports (5555/5556) are available

### Not receiving messages
- Check your `subscribe_to` list
- Verify message types match exactly (case-sensitive)
- Ensure sender is using correct message type

### Shutdown issues
- Always use `is_running()` in loops
- Don't use `sys.exit()` directly
- Let Corona handle the shutdown process

## 🚀 Advanced Topics

### Custom Message Types

Create an enum for organization:

```python
class MessageTypes:
    SENSOR_DATA = "SENSOR_DATA"
    CONTROL_CMD = "CONTROL_CMD"
    STATUS_REQ = "STATUS_REQ"
    STATUS_RSP = "STATUS_RSP"
```

### Async Operations

Use threading for long operations:

```python
def main_loop(is_running):
    def process_async(data):
        result = long_operation(data)
        out_queue.put(SolarFlare(...))
    
    while is_running():
        if not in_queue.empty():
            flare = in_queue.get()
            threading.Thread(target=process_async, args=(flare.payload,)).start()
```

## 📚 API Reference

### CometCore

- `name`: Your Comet's unique identifier
- `subscribe_to`: List of message types to receive
- `in_queue`: Incoming messages appear here
- `out_queue`: Put outgoing messages here
- `on_startup`: Called once at start
- `on_shutdown`: Called once at shutdown
- `main_loop`: Your main logic (receives `is_running` function)

### SolarFlare

- `timestamp`: When the message was created
- `name`: Sender's name
- `type`: Message type (any string)
- `payload`: Message data (must be JSON-serializable)

## 🎉 You're Ready!

1. Copy this template
2. Rename to your Comet's name
3. Edit `comet/src/main.py`
4. Build and distribute!

Welcome to the Sunshine ecosystem! 🌟