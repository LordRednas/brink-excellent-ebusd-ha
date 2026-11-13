# Step by step: Brink Excellent + eBUS Adapter C6 + Home Assistant

🇳🇱 [Nederlandse versie](stap-voor-stap.md) · ⬅️ [Back to README](../README.md)

Exactly how I got it working, including the traps I fell into. 😉

---

## Step 1 – Put the eBUS stick on your network

1. Power the **eBUS Adapter C6** from a **USB charger** (5 V, at least 1 A). The eBUS wires alone don't deliver enough power for Wi-Fi.
2. Connect the adapter to your Wi-Fi via its own access point (see the [adapter documentation](https://adapter.ebusd.eu/)).
3. Find its IP in your router (name like `ebus-xxxxxx`) and give it a **fixed IP** (DHCP reservation).
4. Open `http://<adapter-ip>/`. The **Status** page shows:
   - `ebusd device string: "ens:<adapter-ip>:9999"` ← you'll need this
   - `WiFi station/client: <your SSID>, <ip>, xx% (-xx dBm)`
   - `ebusd connected` and `eBUS signal` (both ❌ for now – that's fine)

## Step 2 – Check the stick's configuration

Under **Configuration** (for me everything was already correct):

| Section | Setting |
|---|---|
| WIFI station/client | **On**, your SSID + password |
| WIFI access point | Off |
| eBUS | **Enhanced protocol (default)** + **TCP** |
| micro-ebusd | **Not used** – so **no token** needed |
| MQTT | **unused** – the ebusd add-on in Home Assistant handles MQTT |
| httpd | "no TLS, no authorization" is fine on a home network |

> ⚠️ **Trap:** if you see a red *"API retrieval failed … Failed to fetch"* at the top, the page didn't load the settings properly. It may show e.g. "WIFI station (off)" while it is actually on. **Do NOT press Save** – you'd really turn Wi-Fi off! Reload the page first.

> 📶 **Wi-Fi:** below about **–75 dBm (≈ 50 %)** the connection drops every minute. It still works, but a better spot or a nearby access point makes it stable.

## Step 3 – ebusd add-on in Home Assistant

1. **Settings → Add-ons → Add-on store → ⋮ → Repositories** → add `https://github.com/LukasGrebe/ha-addons`
2. Install **eBUSd**.
3. **Configuration** tab:
   - **Network / enhanced-protocol adapter** (`network_device`): `ens:<adapter-ip>:9999`
   - leave the rest default (MQTT is linked automatically to the Mosquitto add-on)
4. Enable **Watchdog** and **start** the add-on.
5. The **log** shows:
   ```
   [device notice] transport opened
   [bus notice] bus started with own address 31/36
   [mqtt notice] connection established
   ```
   The stick's status page now says **ebusd connected: yes**. ✅

> ℹ️ Do **not** use the old "ebusd" integration under *Add integration* (it asks for `configuration.yaml`). You only need the **add-on**.

## Step 4 – Connect the stick to the Brink

1. ⚡ **Unplug the Brink from mains power first.**
2. Find the 2-pole screw terminal **"eBus X1"** (that's its name on the Excellent 300; according to the manual it can also be used for OpenTherm).
3. Connect the **2 wires** from the stick. A twisted pair (e.g. 2 wires of a UTP cable) is best.
4. ⚠️ The Brink manual says the connector is **polarity sensitive** in eBUS mode. If you get no signal later, swap the wires.
5. Plug the Brink back in.

## Step 5 – Configure the Brink (menu steps 08 and 09) 🔑

This was **the whole problem** for me. Without it you'll only see *"no signal"*.

**Entering the settings menu** (Brink display, 4 buttons):

1. Press **MENU** → menu overview.
2. Press **MENU** again → the **settings menu** (SET) is active.
3. Use **+** / **–** to choose the **step number**.
4. Press **MENU** to select it.
5. Change the value with **+** / **–**.
6. Press **MENU** to **save** (or **R** to **not** save).
7. Repeat for the next step; press **R** to go back to the normal screen.

**Set these two steps:**

| Step | Name | Set to | Why |
|---|---|---|---|
| **08** | Communication type | **eBUS** | The other option is `Ot` (OpenTherm) – eBUS won't work then. |
| **09** | eBus address | **0** (= master) | 0 = master, 1–9 = slave. **If the Brink is the only device on the bus** (no Air Control, eModule or zone valve), **it must be master.** Mine was on a slave address → no bus voltage and no sync. |

➡️ **Then power-cycle the Brink** (unplug for 10 seconds) so the settings take effect.

> 🕵️ **Tell-tale sign that step 09 is wrong:** right after a Brink reboot the ebusd log shows exactly **2 seconds** of signal, then nothing:
> ```
> [bus notice] signal acquired
> [bus error] signal lost
> ```

## Step 6 – Check with a multimeter (optional)

- Set the meter to **V DC** (V with a straight line). **Not** V AC – that gives nonsense (I got 0.10).
- Measure across the two eBUS wires: roughly **15–24 V**. I measured **20.88 V** once step 09 was 0.
- **0 V**? Check steps 08/09 and power-cycle the Brink.

## Step 7 – Success? 🎉

The ebusd log shows:
```
[bus notice] signal acquired
[main notice] starting initial broadcast scan
[bus notice] new master 37, master count 2
[bus notice] scan 3c: ;ENCON;  ;-;-
[main notice] read scan config file encon/3c..excellent.csv for ID "", SW65535, HW65535
[main notice] found messages: 156 (...)
```
- **ENCON** = Brink, **3c** = eBUS address of the Excellent 300.
- ebusd finds the right configuration **by itself** – **no custom CSV** needed.
- About 140 entities `sensor.heating_ebusd_excellent_*` appear in Home Assistant; they fill in within ~15 minutes.

Then continue with **installing this repo** ([README](../README.md#5-this-repository)) for buttons, airflow, notifications and the dashboard.

## Traps at a glance

| What I saw | Cause / fix |
|---|---|
| Stick: *eBUS signal: no signal* | Wires not connected, step 08 not eBUS, or **step 09 not 0** |
| ~2 s signal after Brink reboot, then lost | **Step 09 → 0** and power-cycle |
| Stick unreachable after connecting | No/weak USB power, or weak Wi-Fi at the unit's location |
| ebusd: `unable to open …:9999` | Stick unreachable (power, Wi-Fi, IP changed) |
| Measured 0.10 or 0.0 | Measuring **V AC** instead of **V DC**, or step 09 wrong |
| `write message excellent FanMode not found` | Use `SetFanModeAlt` (see the scripts in this repo) |
