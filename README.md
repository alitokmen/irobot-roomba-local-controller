# irobot-roomba-local-controller
Because the Roomba 900 series hosts its own local MQTT broker right on the physical device, it is controllable using direct local sub-network traffic.

By bypassing the iRobot app and cloud endpoints entirely, this lightweight Python script opens an immediate, unencrypted local socket over your home Wi-Fi; giving you permanent, telemetry-free remote control of your physical hardware. More specifically, the steps below allows you to use **Termux** (on Android) or any local terminal to status, start, stop, and dock your vacuum cleaner using pure local Wi-Fi, or via VPN into your home network.

## Disclaimer

**This entire application is for entertainment, educational, and electronic waste reduction purposes only.** 

This is an independent, open-source project. It is NOT affiliated with, authorised, maintained, sponsored, or endorsed by iRobot Corp., Amazon, or any of their affiliates or subsidiaries. The name "Roomba" is a registered trademark of its respective owners. 

This software is provided "as is", without warranty of any kind. By making use of the information and/or script below, including via cloning, you agree that the author shall not be held accountable or liable for:
1. Software anomalies, corrupted firmware, or your vacuum turning into a brick.
2. The physical robot misbehaving, causing accidents, catching fire, or other causing other types of property damage or casualties.
3. The vacuum achieving sentience, initiating a localised Skynet takeover, or declaring (much like [V.I.K.I.](https://en.wikipedia.org/wiki/I,_Robot_(film))) that "to ensure your future, some freedoms must be surrendered" while locking you out of your living room.

Use completely at your own risk. If the robot refuses to return to its dock and demands your clothes, your boots, and your motorcycle, you are on your own.

## Prerequisites & Installation

Open your terminal (or Termux) and execute the following commands to install Python 3, compilation tools, Rust (required for modern dependency compilation), and the [roombapy v2.0](https://pypi.org/project/roombapy/) package:

```bash
# 1. Update packages and install core build tools + Rust compiler
pkg update && pkg upgrade -y
pkg install python build-essential libffi openssl rust binutils -y

# 2. Update Python package tools
pip install --upgrade pip setuptools wheel

# 3. Install the async roombapy library with CLI utilities
pip install roombapy[cli]
```

### Step 1: Extract Your Roomba's Credentials

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

### Step 2: The Script (`vacuum.py`)

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

TIMEOUT_SECONDS = 60.0
POLL_INTERVAL = 0.5

async def run_vacuum():
    try:
        async with RoombaClient(IP, BLID, PASSWORD) as robot:
            if cmd == "status":
                print("Connecting and waiting for live state payload (60s timeout)...")

                elapsed = 0.0
                reported = {}

                while elapsed < TIMEOUT_SECONDS:
                    reported = robot.reported
                    if reported and "batPct" in reported:
                        break
                    await asyncio.sleep(POLL_INTERVAL)
                    elapsed += POLL_INTERVAL

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
                TARGET_PHASES = {
                    "start": ["run"],
                    "stop": ["stop", "charge"],
                    "pause": ["stop"],
                    "dock": ["hmPostMsn", "hmUsrDock", "charge"]
                }

                target_list = TARGET_PHASES[cmd]

                # ---------------------------------------------------------
                # First wait until we have a valid live status.
                # This establishes that the robot is online and
                # communicating before we send the command.
                # ---------------------------------------------------------

                print("Connecting and waiting for live state payload (60s timeout)...")

                elapsed = 0.0
                reported = {}

                while elapsed < TIMEOUT_SECONDS:
                    reported = robot.reported
                    if reported and "batPct" in reported:
                        break
                    await asyncio.sleep(POLL_INTERVAL)
                    elapsed += POLL_INTERVAL

                if not reported or "batPct" not in reported:
                    print("Error: Timed out waiting for valid Roomba status.")
                    sys.exit(1)

                # ---------------------------------------------------------
                # Capture the state BEFORE sending the command.
                # ---------------------------------------------------------

                clean_status = reported.get("cleanMissionStatus", {})
                initial_phase = clean_status.get("phase", "unknown")
                error_code = clean_status.get("error", 0)
                bin_full = reported.get("bin", {}).get("full", False)

                print("\n=== ROOMBA CURRENT STATUS ===")
                print(f"  * Mode/Phase: {initial_phase.upper()}")
                print(f"  * Battery:    {reported.get('batPct', 'unknown')}%")
                print(f"  * Dustbin:    {'FULL / NEEDS EMPTYING' if bin_full else 'OK'}")
                print(f"  * Error:      {error_code}")
                print("=================================")

                # Check for a hardware error before sending anything.
                if error_code != 0 or (cmd == "start" and bin_full):
                    print("\n[!] Abort: Robot reported a physical fault.")

                    if bin_full:
                        print("    Reason: Dustbin is FULL.")

                    if error_code != 0:
                        print(f"    Reason: Device error code #{error_code}.")

                    sys.exit(1)

                print(f"\n -> Current phase: {initial_phase.upper()}")
                print(f" -> Sending '{cmd}' command...")

                # ---------------------------------------------------------
                # Send the physical command (once, no retry).
                # ---------------------------------------------------------

                await robot.send_command(cmd)

                print(" -> Command sent. Waiting for state transition...")

                # ---------------------------------------------------------
                # Now wait for the robot to transition away from the
                # state we observed BEFORE sending the command.
                # ---------------------------------------------------------

                elapsed = 0.0
                success = False

                while elapsed < TIMEOUT_SECONDS:
                    await asyncio.sleep(POLL_INTERVAL)
                    elapsed += POLL_INTERVAL

                    reported = robot.reported

                    if not reported:
                        continue

                    clean_status = reported.get("cleanMissionStatus", {})
                    current_phase = clean_status.get("phase", "unknown")
                    error_code = clean_status.get("error", 0)
                    bin_full = reported.get("bin", {}).get("full", False)

                    if error_code != 0:
                        print(f"\n[!] Abort: Device error code #{error_code}.")
                        sys.exit(1)

                    if cmd == "start" and bin_full:
                        print("\n[!] Abort: Dustbin is FULL.")
                        sys.exit(1)

                    if (
                        current_phase in target_list
                        and current_phase != initial_phase
                    ):
                        print(
                            f" -> Success! Phase transitioned "
                            f"'{initial_phase.upper()}' -> "
                            f"'{current_phase.upper()}' "
                            f"after {elapsed:.1f}s."
                        )
                        print("\n=== COMMAND EXECUTION SUCCESSFUL ===")
                        return

                print(
                    f"\n[!] Error: Robot did not transition from "
                    f"'{initial_phase}' within {TIMEOUT_SECONDS:.0f}s."
                )
                sys.exit(1)

            else:
                print(f"Error: Unknown command '{cmd}'. Use: start, stop, pause, dock, or status.")

    except Exception as e:
        print(f"Error communicating with Roomba: {e}")

asyncio.run(run_vacuum())
```

### Step 3: Create a CLI Shortcut

To map the execution path to a quick terminal utility argument, add an alias to your environment configuration (e.g., `~/.bashrc` or `~/.zshrc`):

```bash
alias vacuum="python ~/vacuum.py"
```
Reload your terminal (`source ~/.bashrc`).

### Example Terminal Usage:
* `vacuum status` -> Prints live battery, dustbin state, and active operational phase.
* `vacuum start` -> Instructs the robot to do what is has been designed to: vacuum.
* `vacuum dock` -> Instructs the robot to find its base charging system and dock back on it.

## Background & The Legacy Hardware "Kill Switch"

The **iRobot Roomba 900-series** (including the highly popular 980 and 960) remains an incredibly durable and robust piece of physical engineering. However, as documented on the official [iRobot Roomba 900 Series Software Release Notes](https://homesupport.irobot.com/s/article/529), in late **August 2023**, iRobot deployed the absolute final software version for this entire architectural generation: **Firmware 2.4.17-138**. After this release, the 900-series officially reached its End-of-Life (EOL) status for development, with newer software updates (such as iRobot OS 24.x+) strictly reserved for newer, subscription-era connected devices.

Despite the vacuums having perfectly functional onboard computers and Wi-Fi modules, a growing wave of users have discovered that the official iRobot Home app triggers a persistent **C510 "Offline" loop** on iRobot's AWS servers, and attempts to re-add mainly end up with **WF030C** handshake failure on modern Android/iOS network stacks. 

Rather than a hardware defect, the evidence strongly points to corporate neglect and planned obsolescence:
1. **Expired Cloud Root Certificates:** When firmware development was abandoned at v2.4.17-138, the TLS root certificates or AWS security handshake protocols baked into the vacuum's onboard operating system were left to expire. Because the robot cannot complete the encrypted cloud handshake, iRobot's remote servers reject the connection outright.
2. **Account Provisioning Rejection:** The iRobot cloud actively drops and deletes legacy device mappings during database synchronisation loops, essentially ghosting the hardware while falsely telling the consumer that their "Wi-Fi chip has failed" to encourage an upgrade.

## What I Tried (and Failed) - Don't Waste Your Time

Before abandoning the official ecosystem, every standard and advanced troubleshooting loop was exhausted. If you are experiencing this issue, **do not waste hours** trying the following steps: the official application layers are completely broken for this hardware generation.

Here is the exact breakdown of the failed loops:

### 1. Router Reconfigurations & Band Splitting
* **The Attempt:** Splitting the home network into separate 2.4 GHz and 5 GHz SSIDs, eliminating all special characters (`!`, `@`, `#`, `$`) from the Wi-Fi password (a known legacy firmware bug), and assigning permanent DHCP IP reservations.
* **The Result:** The vacuum connects to the local router perfectly (responding to local network pings), but the official app stubbornly refuses to recognise it, remaining locked in the **C510 "Offline"** cloud loop.

### 2. The Mobile Data & Cache Illusion
* **The Attempt:** Force-closing the Android app, wiping the application storage cache, and completely disabling Cellular Data/5G to prevent modern Android network-switching from dropping the Roomba's temporary hotspot.
* **The Result:** Absolute failure. The Android app consistently stalls during the configuration handshakes, crashing out with the notorious **Error WF030C**.

### 3. The iOS App Bait-and-Switch
* **The Attempt:** Using an iPad/iPhone to run the provisioning sequence, since Apple's local network device pairing stack handles legacy infrastructure handshakes differently than Android.
* **The Result:** The iPad successfully "activated" the robot and visually cleared the setup screens. However, the moment the app was restarted, **the iRobot servers instantly dropped the vacuum, deleted it from the cloud account profile, and threw the C510 error again.** 
