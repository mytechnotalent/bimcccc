![image](https://github.com/mytechnotalent/bimcccc/blob/main/bimcccc.png?raw=true)

## FREE Reverse Engineering Self-Study Course [HERE](https://github.com/mytechnotalent/Reverse-Engineering-Tutorial)

<br><br>

# BLE Interactive Meshtastic Custom Channel Chat Client
BLE Interactive Meshtastic Custom Channel Chat Client which chats on a Custom Channel.

<br><br>

### STEP 1: `python3 -m venv venv`
### STEP 2: `source venv/bin/activate`
### STEP 3: `pip install -r requirements.txt`
### STEP 4: `meshtastic --ble-scan`
### STEP 5: `./bimcccc.py <BLE_DEVICE_ADDRESS>`

### SOURCE
```python
#!/usr/bin/env python3

"""
BLE Interactive Meshtastic Custom Channel Chat Client 0.1.0

Usage:
    python bimcccc.py <BLE_DEVICE_ADDRESS>

This script sends and receives text messages over the BLE interface.
It uses PyPubSub to subscribe to text messages.
"""

import sys
import time
import asyncio
from pubsub import pub
from meshtastic.ble_interface import BLEInterface, BLEClient

# custom channel configuration
CUSTOM_CHANNEL_NAME = "DC540"
CUSTOM_CHANNEL_PSK = "OVRpanBCYjZ2WmVIZTRheVlSZDZZWGxUcElFNFRSaWo="


def custom_find_device(address):
    """
    Scans for BLE devices and matches against the provided address.
    """
    devices = BLEInterface.scan()
    print("Discovered Devices:")
    for d in devices:
        print(f"  Address: {d.address}, Name: {d.name}")
    addr_lower = address.lower()
    for d in devices:
        if d.address.lower() == addr_lower:
            return d
    raise Exception(f"No BLE peripheral with address '{address}' found. Try --ble-scan.")


def custom_connect(self, address=None):
    """
    Connects to a BLE device by skipping the scanning process if an address is given.
    """
    if address:
        client = BLEClient(address, disconnected_callback=lambda _: self.close)
        client.connect()  # synchronous connect (BleakClient.connect)
        client.discover()  # discover services/characteristics
        return client
    else:
        return _original_connect(self, address)


def onReceive(packet=None, interface=None):
    """
    Handles incoming text messages from the custom channel.
    """
    if not packet:
        return
    decoded = packet.get("decoded", {})
    if "text" in decoded:
        sender = packet.get("fromId", "unknown")
        message = decoded["text"]
        print(f"\n{sender}: {message}")
        print("Ch1> ", end="", flush=True)


async def main():
    """
    Main function for BLE-based Meshtastic custom chat client.
    """
    if len(sys.argv) < 2:
        print("Usage: python bimcc_custom.py <BLE_DEVICE_ADDRESS>")
        sys.exit(1)
    
    address = sys.argv[1].strip()
    print(f"Attempting to connect to BLE device at address: {address}")

    try:
        ble_iface = BLEInterface(address=address)
        print("Connected to BLE device!")
        time.sleep(2)
    except Exception as e:
        print("Error initializing BLE interface:", e)
        sys.exit(1)

    # Configure custom channel
    try:
        print("Setting custom Channel 1 config (DC540 + custom PSK)...")
        node = ble_iface.localNode
        ch1 = node.channels[1]  # channel index 1
        ch1.name = CUSTOM_CHANNEL_NAME
        ch1.psk = CUSTOM_CHANNEL_PSK  # base64 for 256-bit key
        ch1.usePreset = False  # disable built-in presets
        node.writeChannel(1)
        ble_iface.writeConfig()
        print("Channel 1 set to DC540 with custom PSK.\n")
    except Exception as e:
        print("Error configuring Channel 1:", e)

    print("BLE Interactive Meshtastic Custom Channel Chat Client 0.1.0")
    print("-----------------------------------------------------------")
    print("Type your message and press Enter to send.")
    print("Press Ctrl+C to exit...\n")

    loop = asyncio.get_running_loop()
    try:
        while True:
            msg = await loop.run_in_executor(None, input, "Ch1> ")
            if msg:
                ble_iface.sendText(msg, channelIndex=1)
                await asyncio.sleep(0.1)
    except KeyboardInterrupt:
        print("\nExiting...")
    finally:
        ble_iface.close()
        sys.exit(0)


if __name__ == "__main__":
    BLEInterface.find_device = custom_find_device
    _original_connect = BLEInterface.connect
    BLEInterface.connect = custom_connect

    pub.subscribe(onReceive, "meshtastic.receive.text")

    asyncio.run(main())
```

<br>

## License
[Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0)
