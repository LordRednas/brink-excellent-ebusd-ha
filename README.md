# Brink Renovent Excellent + ebusd + Home Assistant

🇳🇱 [Nederlandse versie](README.nl.md) · 🪜 **[Step-by-step setup guide](docs/step-by-step.md)** (adapter, ebusd add-on, Brink menu 08/09)

Fully local monitoring **and control** of a **Brink Renovent Excellent 300** heat-recovery ventilation unit (WTW / HRV) in Home Assistant, using an **eBUS Adapter C6** stick and the **ebusd** add-on. No Brink cloud, no eModule, no Air Control panel needed.

What you get:

- 🌬️ Read **and set** the fan mode (Away / 1 / 2 / 3)
- ☀️ Read and set the **bypass** (Auto / Closed / Open) and the bypass temperature
- 💨 **Actual airflow** (m³/h) and fan speeds (rpm)
- 🌡️ Extract-air and outside temperatures, plus an **estimated supply-air temperature**
- 🧽 Filter status, filter age in days, a **filter reset** button and push notifications
- 🤖 Blueprints: **shower boost** (via a water-flow sensor), **night mode + summer night cooling**, **vacation mode**, **filter notification**
- 📊 A ready-made dashboard view with graphs

> Tested with: Brink Renovent Excellent 300 (eBUS slave address `3c`, ENCON), eBUS Adapter Stick C6 over Wi-Fi (enhanced protocol), ebusd add-on 26.1, Home Assistant OS.
> Other Excellent models (400, 450, Plus) probably work too (the 400 uses slave `7c` → circuit name may differ). Feedback welcome!

---

## 1. Hardware

| Part | Notes |
|---|---|
| eBUS Adapter **C6** (stick or shield) | Connected to your Wi-Fi; powered by **USB** (the eBUS wires alone are not enough) |
| 2-wire cable | Twisted pair (e.g. 2 wires of a UTP cable) works fine |
| Brink Renovent Excellent | eBUS connector **X1** (2-pole screw terminal) |

⚠️ **Disconnect the Brink from mains power before opening the terminal cover.**
⚠️ The Brink manual says the eBUS connector is **polarity sensitive** – if you get no signal, swap the two wires.

## 2. Settings on the Brink (installer menu) – the important part!

| Step | Name | Value | Why |
|---|---|---|---|
| **08** | Communication | **eBUS** | Not `Ot` (OpenTherm) |
| **09** | Bus address | **0 = master** | 🔑 **If the Brink is the only device on the bus, it must be master.** Otherwise there is no bus voltage/sync and the adapter shows *“no signal”* (you may only see ~2 seconds of signal right after a reboot). |

After changing step 09: **power-cycle the Brink**.

You can verify with a multimeter (**V DC!**) across the two eBUS wires: roughly **15–24 V** (I measured 20.9 V).

## 3. The eBUS adapter

In the adapter web UI (`http://<adapter-ip>/`):

- **eBUS: Enhanced protocol + TCP** (default) – leave **micro-ebusd off**, no token needed.
- MQTT on the adapter: **unused** (ebusd in Home Assistant does that).
- Give the adapter a **fixed IP** (DHCP reservation in your router).
- Weak Wi-Fi (below ~ –75 dBm) causes reconnects every minute. Better placement or an access point helps.

The status page should show **ebusd connected: yes** and **eBUS signal: acquired**.

## 4. ebusd add-on

1. Add the repository `https://github.com/LukasGrebe/ha-addons` and install **eBUSd**.
2. Configuration:
   - `network_device`: `ens:<adapter-ip>:9999`
   - leave the rest default (MQTT is picked up automatically from the Mosquitto add-on)
3. Start it. In the log you should see:

```
[bus notice] signal acquired
[bus notice] scan 3c: ;ENCON;  ;-;-
[main notice] read scan config file encon/3c..excellent.csv
```

🎉 **No custom CSV needed** – the official ebusd configuration already knows the Excellent, including the **write** messages (`SetFanModeAlt`, `BypassMode`, `BypassTemp`, `ResetNotifications`, …). They just don’t show up as entities in HA by default, so this repo uses scripts that publish to `ebusd/excellent/<message>/set`.

You will get ~140 entities named `sensor.heating_ebusd_excellent_*` (they fill in during the first ~15 minutes).

## 5. This repository

```
packages/brink_wtw.yaml          scripts, MQTT airflow sensors, helpers, template sensors, automation
blueprints/automation/brink_wtw/ shower_boost, night_mode, vacation_mode, filter_notification
dashboard/wtw-view.yaml          dashboard view (Mushroom + ApexCharts)
```

### Install

1. Copy `packages/brink_wtw.yaml` to `/config/packages/` and enable packages in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Restart Home Assistant.
3. Import the blueprints (Settings → Automations → Blueprints → Import) using the raw GitHub URLs of the files in `blueprints/`.
4. Add the dashboard view from `dashboard/wtw-view.yaml` (needs **Mushroom** and **ApexCharts** from HACS).
5. Set `input_datetime.brink_wtw_filter_last_reset` to the date you last cleaned the filters.

## 6. Good to know

- **Fan mode**: `SetFanModeAlt` is the message the Brink Air Control panel sends; it stays active (unlike `SetFanMode`, which resets after a few minutes). A wall switch on the unit can still override it – last command wins.
- **Bypass mode** and **bypass temperature** are *settings* (menu steps 05/07), not temporary states. Put the bypass back on **Auto** when you’re done.
- **Bypass logic (from the manual):** opens only when outside > 10 °C **and** outside < inside **and** inside > bypass temperature (step 05, default 22 °C). Closes below *bypass temperature – hysteresis* (default 20 °C). If your house is around 22–23 °C you may get unwanted cool air on mild days; raising the bypass temperature to **24 °C** fixes that.
- **Actual airflow / fan speed** are not polled by ebusd by default. The script `brink_wtw_poll_airflow` sends `?1` (poll priority 1) to `ebusd/excellent/<message>/get`; ebusd forgets this on restart, so an automation re-sends it.
- The Excellent has **no supply-air temperature sensor**. The estimated value uses the 95 % efficiency from the Brink manual; real-world efficiency is usually lower. For a real measurement, put a cheap Zigbee temperature sensor at a supply valve.
- The **filter reset** also resets the Brink’s internal filter counter – only press it after actually cleaning/replacing the filters.

## 7. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Adapter: *eBUS signal: no signal* | Brink step 08 not on eBUS, **step 09 not 0 (master)**, wrong terminals, loose wire, or polarity |
| Signal for ~2 s after Brink reboot, then lost | **Step 09 → 0 (master)** and power-cycle |
| 0 V measured | Measure in **V DC**; check step 08/09 |
| ebusd log: `unable to open …:9999` | Adapter not reachable: power, Wi-Fi strength, IP changed |
| `write message excellent FanMode not found` | Use `SetFanModeAlt` instead of `FanMode` |

## Credits

- [john30/ebusd](https://github.com/john30/ebusd) and [ebusd-configuration](https://github.com/john30/ebusd-configuration)
- [LukasGrebe/ha-addons](https://github.com/LukasGrebe/ha-addons) (ebusd add-on)
- [christiaanderidder/brink-renovent-hru](https://github.com/christiaanderidder/brink-renovent-hru) and [pvyleta/ebusd-brink-hru](https://github.com/pvyleta/ebusd-brink-hru) for the Brink eBUS research

*Brink is a trademark of its owner. This project is independent and not affiliated with Brink Climate Systems. Use at your own risk.*

## License

MIT
