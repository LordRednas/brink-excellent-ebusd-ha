# Stap voor stap: Brink Excellent + eBUS Adapter C6 + Home Assistant

🇬🇧 [English version](step-by-step.md) · ⬅️ [Terug naar README](../README.nl.md)

Dit is precies hoe ik het werkend heb gekregen, inclusief de valkuilen waar ik in trapte. 😉

---

## Stap 1 – De eBUS-stick op je netwerk zetten

1. Steek de **eBUS Adapter C6** in een **USB-voeding** (een gewone 5V-lader van minimaal 1A). De eBUS-draadjes alleen leveren niet genoeg stroom voor de wifi.
2. Verbind de adapter met je wifi. Dat gaat via het eigen wifi-accesspoint van de adapter (zie de [handleiding van de adapter](https://adapter.ebusd.eu/)).
3. Zoek het IP-adres op in je router (de naam is iets als `ebus-xxxxxx`) en geef de adapter daar een **vast IP-adres** (DHCP-reservering).
4. Open `http://<ip-adapter>/` in je browser. Op de **statuspagina** zie je:
   - `ebusd device string: "ens:<ip-adapter>:9999"` ← deze heb je straks nodig
   - `WiFi station/client: <jouw wifi>, <ip>, xx% (-xx dBm)`
   - `ebusd connected` en `eBUS signal` (die staan nu nog op ❌, dat is normaal)

## Stap 2 – De instellingen van de stick controleren

Ga naar **Configuration** en controleer het volgende. Bij mij was alles al goed en hoefde ik niets aan te passen.

| Onderdeel | Instelling |
|---|---|
| WIFI station/client | **On**, met jouw SSID en wachtwoord |
| WIFI access point | Off |
| eBUS | **Enhanced protocol (default)** + **TCP** |
| micro-ebusd | **Niet** gebruiken, en je hebt dus ook **geen token** nodig |
| MQTT | **unused**, want dat doet de ebusd-add-on in Home Assistant |
| httpd | Mag op "no TLS, no authorization" blijven staan in je thuisnetwerk |

> ⚠️ **Valkuil:** staat er bovenaan een rode melding *"API retrieval failed … Failed to fetch"*, dan heeft de pagina de instellingen niet goed geladen. Je ziet dan bijvoorbeeld "WIFI station (off)" terwijl die in werkelijkheid aan staat. **Druk dan NIET op Save**, anders zet je je wifi echt uit! Herlaad eerst de pagina.

> 📶 **Wifi:** onder ongeveer **–75 dBm (≈ 50%)** valt de verbinding om de minuut even weg. Het werkt dan nog wel, maar met een betere plek of een access point dichtbij is het stabieler.

## Stap 3 – De ebusd-add-on in Home Assistant

1. Ga naar **Instellingen → Add-ons → Add-on store → ⋮ → Repositories** en voeg toe: `https://github.com/LukasGrebe/ha-addons`
2. Installeer **eBUSd**.
3. Ga naar het tabblad **Configuratie** en vul in:
   - **Network / enhanced-protocol adapter** (`network_device`): `ens:<ip-adapter>:9999`
   - laat de rest op standaard staan (MQTT wordt automatisch gekoppeld aan de Mosquitto-add-on)
4. Zet **Watchdog** aan en **start** de add-on.
5. In de **log** zie je:
   ```
   [device notice] transport opened
   [bus notice] bus started with own address 31/36
   [mqtt notice] connection established
   ```
   Op de statuspagina van de stick staat nu **ebusd connected: yes**. ✅

> ℹ️ Gebruik **niet** de oude "ebusd"-integratie die je vindt via *Integratie toevoegen* (die vraagt om `configuration.yaml`). Je hebt alleen de **add-on** nodig.

## Stap 4 – De stick aansluiten op de Brink

1. ⚡ **Haal eerst de stekker van de Brink uit het stopcontact.**
2. Zoek de tweepolige schroefconnector **"eBus X1"** (bij de Excellent 300 heet hij zo; volgens de handleiding kan deze connector ook voor OpenTherm gebruikt worden).
3. Sluit de **2 draadjes** van de stick daarop aan. Gebruik bij voorkeur een getwist paar, bijvoorbeeld 2 aders uit een UTP-kabel.
4. ⚠️ Volgens de Brink-handleiding is de connector bij eBUS **polariteitsgebonden**. Krijg je straks geen signaal, draai de draadjes dan om.
5. Steek de stekker van de Brink er weer in.

## Stap 5 – De Brink instellen (menu 08 en 09) 🔑

Dit was bij mij **het hele probleem**. Zonder deze stap zie je alleen *"no signal"*.

**Zo kom je in het instelmenu** (op het display van de Brink, met de 4 toetsen):

1. Druk op **MENU**: je komt in het menuoverzicht.
2. Druk nog een keer op **MENU**: het **instelmenu** is nu actief.
3. Kies met **+** / **–** het **stapnummer** (zie hieronder).
4. Druk op **MENU** om dat stapnummer te selecteren.
5. Pas de waarde aan met **+** / **–**.
6. Druk op **MENU** om **op te slaan**. Met **R** sla je juist **niet** op.
7. Herhaal dit voor de volgende stap, en druk op **R** om terug te gaan naar het normale scherm.

**Deze twee stappen moet je instellen:**

| Stap | Naam | Zet op | Uitleg |
|---|---|---|---|
| **08** | Type communicatie | **eBUS** | De andere optie is `Ot` (OpenTherm). Op `Ot` werkt eBUS niet. |
| **09** | eBus-adres | **0** (= master) | 0 = master, 1 t/m 9 = slave. **Is de Brink de enige op de bus** (geen Air Control, eModule of zoneklep)? **Dan moet hij master zijn.** Bij mij stond hij op een slave-adres, en daardoor was er geen busspanning en geen sync. |

➡️ **Maak de Brink daarna even spanningsloos** (stekker 10 seconden eruit), zodat de instellingen actief worden.

> 🕵️ **Herkenningsteken van stap 09 verkeerd:** in de ebusd-log zie je vlak na een herstart van de Brink precies **2 seconden** signaal, en daarna niets meer:
> ```
> [bus notice] signal acquired
> [bus error] signal lost
> ```

## Stap 6 – Controleren met een multimeter (optioneel)

- Zet je multimeter op **V DC** (de V met een rechte streep). **Niet** op V AC, want dan meet je onzin (bij mij gaf dat 0,10).
- Meet tussen de twee eBUS-draden: ongeveer **15–24 V**. Ik mat **20,88 V** nadat stap 09 op 0 stond.
- Meet je **0 V**? Controleer dan stap 08 en 09 en herstart de Brink.

## Stap 7 – Is het gelukt? 🎉

In de ebusd-log zie je:
```
[bus notice] signal acquired
[main notice] starting initial broadcast scan
[bus notice] new master 37, master count 2
[bus notice] scan 3c: ;ENCON;  ;-;-
[main notice] read scan config file encon/3c..excellent.csv for ID "", SW65535, HW65535
[main notice] found messages: 156 (...)
```
- **ENCON** = Brink, en **3c** = het eBUS-adres van de Excellent 300.
- ebusd vindt **zelf** de juiste configuratie (`3c..excellent.csv`). Je hoeft **geen eigen CSV** toe te voegen.
- In Home Assistant verschijnen ongeveer 140 entiteiten `sensor.heating_ebusd_excellent_*`. Die vullen zich in ongeveer 15 minuten.

Ga daarna verder met de **installatie van deze repo** (zie [README](../README.nl.md#5-installatie)) voor de knoppen, het debiet, de meldingen en het dashboard.

## Valkuilen op een rij

| Wat ik zag | Oorzaak / oplossing |
|---|---|
| Stick: *eBUS signal: no signal* | Draadjes nog niet aangesloten, stap 08 staat niet op eBUS, of **stap 09 is niet 0** |
| ~2 s signaal na een herstart van de Brink, daarna weg | **Zet stap 09 op 0** en maak de Brink spanningsloos |
| Stick niet bereikbaar na het aansluiten | Geen of zwakke USB-voeding, of zwakke wifi op de plek van de Brink |
| ebusd: `unable to open …:9999` | De stick is niet bereikbaar (stroom, wifi, ander IP-adres) |
| 0,10 of 0,0 gemeten | Je meet in **V AC** in plaats van **V DC**, of stap 09 staat niet goed |
| `write message excellent FanMode not found` | Gebruik `SetFanModeAlt` (zie de scripts in deze repo) |
