# irobot-roomba-local-controller
This repository allows you to use **Termux** (on Android) or any local terminal to status, start, stop, and dock your vacuum cleaner using pure local Wi-Fi, bypassing the iRobot / AWS cloud infrastructure.

## Disclaimer

**This entire application is for entertainment, educational, and electronic waste reduction purposes only.** 

This is an independent, open-source project. It is NOT affiliated with, authorized, maintained, sponsored, or endorsed by iRobot Corp., Amazon, or any of their affiliates or subsidiaries. The name "Roomba" is a registered trademark of its respective owners. 

This software is provided "as is", without warranty of any kind. By cloning this repository, you agree that the author shall not be held accountable or liable for:
1. Software anomalies, corrupted firmware, or your vacuum turning into a brick.
2. The physical robot misbehaving, causing accidents, catching fire, or other causing other types of property damage or casualties.
3. The vacuum achieving sentience, initiating a localized Skynet takeover, or declaring —much like V.I.K.I.— that "to ensure your future, some freedoms must be surrendered" while locking you out of your living room.

Use completely at your own risk. If the robot refuses to return to its dock and demands your clothes, your boots, and your motorcycle, you are on your own.

## Background & The Legacy Hardware "Kill Switch"

The **iRobot Roomba 900-series** (including the highly popular 980 and 960) remains an incredibly durable and robust piece of physical engineering. However, as documented on the official [iRobot Roomba 900 Series Software Release Notes](https://homesupport.irobot.com/s/article/529), in late **August 2023**, iRobot deployed the absolute final software version for this entire architectural generation: **Firmware 2.4.17-138**. After this release, the 900-series officially reached its End-of-Life (EOL) status for development, with newer software updates (such as iRobot OS 24.x+) strictly reserved for newer, subscription-era connected devices.

### The Cloud Blackout (Errors C510 & WF030C)

Despite the vacuums having perfectly functional onboard computers and Wi-Fi modules, a growing wave of users have discovered that the official iRobot Home app triggers a persistent **C510 "Offline" loop** on iRobot's AWS servers, and attempts to re-add mainly end up with **WF030C** handshake failure on modern Android/iOS network stacks. 

Rather than a hardware defect, the evidence strongly points to corporate neglect and planned obsolescence:
1. **Expired Cloud Root Certificates:** When firmware development was abandoned at v2.4.17-138, the TLS root certificates or AWS security handshake protocols baked into the vacuum's onboard operating system were left to expire. Because the robot cannot complete the encrypted cloud handshake, iRobot's remote servers reject the connection outright.
2. **Account Provisioning Rejection:** The iRobot cloud actively drops and deletes legacy device mappings during database synchronization loops, essentially ghosting the hardware while falsely telling the consumer that their "Wi-Fi chip has failed" to encourage an upgrade.

### The Solution: 100% Local Wi-Fi Control

**Your Roomba is not dead.** This repository serves as a public workaround to save these premium machines from landfill. 

Because the Roomba 900 series hosts its own local MQTT broker right on the physical device, it remains completely responsive to direct local sub-network traffic. By bypassing the iRobot app and cloud endpoints entirely, this lightweight Python script opens an immediate, unencrypted local socket over your home Wi-Fi—giving you permanent, telemetry-free remote control of your physical hardware.

## What I Tried (and Failed) — Don't Waste Your Time

Before abandoning the official ecosystem, every standard and advanced troubleshooting loop was exhausted. If you are experiencing this issue, **do not waste hours** trying the following steps—the official application layers are completely broken for this hardware generation.

Here is the exact breakdown of the failed loops:

### 1. Router Reconfigurations & Band Splitting
* **The Attempt:** Splitting the home network into separate 2.4 GHz and 5 GHz SSIDs, eliminating all special characters (`!`, `@`, `#`, `$`) from the Wi-Fi password (a known legacy firmware bug), and assigning permanent DHCP IP reservations.
* **The Result:** The vacuum connects to the local router perfectly (responding to local network pings), but the official app stubbornly refuses to recognize it, remaining locked in the **C510 "Offline"** cloud loop.

### 2. The Mobile Data & Cache Illusion
* **The Attempt:** Force-closing the Android app, wiping the application storage cache, and completely disabling Cellular Data/5G to prevent modern Android network-switching from dropping the Roomba's temporary hotspot.
* **The Result:** Absolute failure. The Android app consistently stalls during the configuration handshakes, crashing out with the notorious **Error WF030C**.

### 3. The iOS App Bait-and-Switch
* **The Attempt:** Using an iPad/iPhone to run the provisioning sequence, since Apple's local network device pairing stack handles legacy infrastructure handshakes differently than Android.
* **The Result:** The iPad successfully "activated" the robot and visually cleared the setup screens. However, the moment the app was restarted, **the iRobot servers instantly dropped the vacuum, deleted it from the cloud account profile, and threw the C510 error again.** 

## Prerequisites & Installation

Open your terminal (or Termux) and execute the following commands to install Python 3, compilation tools, Rust (required for modern dependency compilation), and the roombapy v2.0 package:

```bash
# 1. Update packages and install core build tools + Rust compiler
pkg update && pkg upgrade -y
pkg install python build-essential libffi openssl rust binutils -y

# 2. Update Python package tools
pip install --upgrade pip setuptools wheel

# 3. Install the async roombapy library with CLI utilities
pip install roombapy[cli]
```

## Step 1: Extract Your Roomba's Credentials

Your Roomba does not require a cloud connection to speak to your local network, but it does require its unique local credentials (**BLID** and **Password**).

1. Place your Roomba on its charging dock.
2. Press and **hold the physical HOME button** on the vacuum until it plays a short tune and the Wi-Fi icon flashes.
3. Quickly run the discovery tool in your terminal to find its IP and BLID:
   ```bash
   roombapy discover
   ```
4. To extract the hidden password string while the Wi-Fi light is flashing, target the IP explicitly:
   ```bash
   roombapy discover <ROBOT_IP>
   ```
*Copy and save the IP, BLID, and Password.*

## Step 2: The Script (`vacuum.py`)

Create a script file named `vacuum.py` and populate it with the following code. Replace `YOUR_IP`, `YOUR_BLID`, and `YOUR_PASSWORD` with your extracted credentials:

```python
import sys
import asyncio
from roombapy import RoombaClient

if len(sys.argv) < 2:
    print("Usage: vacuum [start|stop|pause|dock|status]")
    sys.exit(1)

cmd = sys.argv[1].lower()

IP = "YOUR_IP"
BLID = "YOUR_BLID"
PASSWORD = "YOUR_PASSWORD"

async def run_vacuum():
    try:
        async with RoombaClient(IP, BLID, PASSWORD) as robot:
            if cmd == "status":
                print("Connecting and waiting for live state payload (60s timeout)...")
                
                timeout_seconds = 60.0
                poll_interval = 0.2
                elapsed = 0.0
                reported = {}
                
                while elapsed < timeout_seconds:
                    reported = robot.reported
                    if reported and "batPct" in reported:
                        break
                    await asyncio.sleep(poll_interval)
                    elapsed += poll_interval
                
                if reported and "batPct" in reported:
                    phase = reported.get("cleanMissionStatus", {}).get("phase", "unknown")
                    battery = reported.get("batPct", "unknown")
                    bin_full = reported.get("bin", {}).get("full", False)
                    
                    print("\n=== ROOMBA CURRENT STATUS ===")
                    print(f"  * Mode/Phase: {phase.upper()}")
                    print(f"  * Battery:    {battery}%")
                    print(f"  * Dustbin:    {'FULL / NEEDS EMPTYING' if bin_full else 'OK'}")
                    print("=================================")
                else:
                    print("Error: Timed out or received incomplete state data from Roomba.")

            elif cmd in ["start", "stop", "pause", "dock"]:
                if cmd == "start": await robot.send_command("start")
                elif cmd == "stop": await robot.send_command("stop")
                elif cmd == "pause": await robot.send_command("pause")
                elif cmd == "dock": await robot.send_command("dock")
                print(f"Sent local '{cmd}' command successfully.")
            else:
                print(f"Error: Unknown command '{cmd}'. Use: start, stop, pause, dock, or status.")
                
    except Exception as e:
        print(f"Error communicating with Roomba: {e}")

asyncio.run(run_vacuum())
```

## Step 3: Create a CLI Shortcut

To map the execution path to a quick terminal utility argument, add an alias to your environment configuration (e.g., `~/.bashrc` or `~/.zshrc`):

```bash
alias vacuum="python ~/vacuum.py"
```
Reload your terminal (`source ~/.bashrc`).

### Example Terminal Usage:
* `vacuum status` -> Prints live battery, dustbin state, and active operational phase.
* `vacuum start` -> Fires up the local cleaning engine.
* `vacuum dock` -> Instructs the robot to find its base charging system.
