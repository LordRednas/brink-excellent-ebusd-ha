# Brink Renovent Excellent + ebusd + Home Assistant

🇬🇧 [English version](README.md) · 🪜 **[Stap-voor-stap-handleiding](docs/stap-voor-stap.md)** (adapter, ebusd-add-on, Brink-menu 08/09)

Volledig lokaal uitlezen **én bedienen** van een **Brink Renovent Excellent 300** WTW in Home Assistant, met een **eBUS Adapter C6**-stick en de **ebusd**-add-on. Zonder Brink-cloud, zonder eModule en zonder Air Control-paneel.

Wat je krijgt:

- 🌬️ De stand uitlezen **en instellen** (Afwezig / 1 / 2 / 3)
- ☀️ De **bypass** (Auto / Dicht / Open) en de bypasstemperatuur uitlezen en instellen
- 💨 Het **actuele luchtdebiet** (m³/h) en de toerentallen van de ventilatoren (rpm)
- 🌡️ De temperatuur van de afgezogen lucht en de buitenlucht, plus een **geschatte toevoertemperatuur**
- 🧽 De filterstatus, de filterleeftijd in dagen, een **resetknop** en pushmeldingen
- 🤖 Blueprints: **douche-boost** (via een watermeter), **nachtstand en zomer-nachtkoeling**, **vakantiemodus** en **filtermelding**
- 📊 Een kant-en-klare dashboardweergave met grafieken

> Getest met: Brink Renovent Excellent 300 (eBUS-slave `3c`, ENCON), eBUS Adapter Stick C6 via wifi (enhanced protocol), ebusd-add-on 26.1 en Home Assistant OS.

---

## 1. Hardware

| Onderdeel | Opmerking |
|---|---|
| eBUS Adapter **C6** (stick of shield) | Op je wifi aangesloten, met **USB-voeding** (de eBUS-draadjes alleen leveren niet genoeg stroom) |
| Tweeaderige kabel | Een getwist paar, bijvoorbeeld 2 aders uit een UTP-kabel |
| Brink Renovent Excellent | eBUS-connector **X1** (tweepolige schroefconnector) |

⚠️ **Haal eerst de stekker van de Brink eruit** voordat je de klemmenkap opent.
⚠️ Volgens de Brink-handleiding is de eBUS-connector **polariteitsgebonden**. Krijg je geen signaal, draai dan de draadjes om.

## 2. Instellingen op de Brink (instelmenu): dit is het belangrijkste!

| Stap | Naam | Waarde | Waarom |
|---|---|---|---|
| **08** | Communicatie | **eBUS** | Niet `Ot` (OpenTherm) |
| **09** | Busadres | **0 = master** | 🔑 **Is de Brink de enige op de bus, dan moet hij master zijn.** Anders krijg je geen busspanning of sync, en geeft de adapter *"no signal"*. Je ziet dan hooguit ongeveer 2 seconden signaal direct na een herstart. |

Na het wijzigen van stap 09: **de Brink even spanningsloos maken**.

Controleren kan met een multimeter (**V DC!**) tussen de twee eBUS-draden: ongeveer **15–24 V** (ik mat 20,9 V).

## 3. De eBUS-adapter

In de webinterface van de adapter (`http://<ip-adapter>/`):

- **eBUS: Enhanced protocol + TCP** (standaard). Laat **micro-ebusd uit**; je hebt geen token nodig.
- Zet MQTT op de adapter op **unused**, want dat regelt ebusd in Home Assistant.
- Geef de adapter een **vast IP-adres** (een DHCP-reservering in je router).
- Bij zwakke wifi (slechter dan ongeveer –75 dBm) maakt hij elke minuut opnieuw verbinding.

Op de statuspagina hoort **ebusd connected: yes** en **eBUS signal: acquired** te staan.

## 4. De ebusd-add-on

1. Voeg de repository `https://github.com/LukasGrebe/ha-addons` toe en installeer **eBUSd**.
2. Configuratie: `network_device` = `ens:<ip-adapter>:9999`. De rest kun je op standaard laten.
3. Start de add-on. In de log zie je dan:

```
[bus notice] signal acquired
[bus notice] scan 3c: ;ENCON;  ;-;-
[main notice] read scan config file encon/3c..excellent.csv
```

🎉 **Je hebt geen eigen CSV nodig.** De officiële ebusd-configuratie kent de Excellent al, inclusief de **schrijf**-opdrachten (`SetFanModeAlt`, `BypassMode`, `BypassTemp`, `ResetNotifications` en meer). Die verschijnen alleen niet automatisch als entiteit in HA, daarom gebruikt deze repo scripts die naar `ebusd/excellent/<bericht>/set` publiceren.

## 5. Installatie

1. Kopieer `packages/brink_wtw.yaml` naar `/config/packages/` en zet packages aan in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Herstart Home Assistant.
3. Importeer de blueprints uit `blueprints/` (Instellingen → Automatiseringen → Blueprints → Importeren).
4. Voeg de dashboardweergave uit `dashboard/wtw-view.yaml` toe. Daarvoor heb je **Mushroom** en **ApexCharts** uit HACS nodig.
5. Zet `input_datetime.brink_wtw_filter_last_reset` op de datum waarop je de filters voor het laatst schoongemaakt hebt.

## 6. Goed om te weten

- **De stand:** `SetFanModeAlt` is het bericht dat het Air Control-paneel ook stuurt, en het blijft actief. Een standenschakelaar aan de muur kan het wel overschrijven: de laatste opdracht wint.
- **De bypassmodus en de bypasstemperatuur** zijn *instellingen* (stap 07 en 05). Zet de bypass na gebruik weer op **Auto**.
- **Wanneer gaat de bypass open?** Alleen als het buiten > 10 °C is **én** buiten < binnen **én** binnen > de bypasstemperatuur (standaard 22 °C). Is je huis ongeveer 22–23 °C, dan blaast hij op frisse dagen koele lucht naar binnen. Zet de bypasstemperatuur dan op **24 °C**.
- **Het actuele luchtdebiet en de toerentallen** pollt ebusd niet uit zichzelf. Het script `brink_wtw_poll_airflow` regelt dat, en een automatisering doet het opnieuw na een herstart.
- **De toevoertemperatuur** is een **schatting** op basis van de 95% uit de handleiding; in de praktijk is het rendement meestal lager. Een Zigbee-temperatuursensor bij een toevoerventiel geeft de echte waarde.
- **De filterreset** zet ook de interne filterteller van de Brink op nul. Druk er dus pas op na het schoonmaken of vervangen van de filters.

## 7. Problemen oplossen

| Symptoom | Waarschijnlijke oorzaak |
|---|---|
| *eBUS signal: no signal* | Stap 08 staat niet op eBUS, **stap 09 is niet 0 (master)**, verkeerde klemmen, een los draadje of de polariteit |
| ~2 s signaal na een herstart van de Brink, daarna weg | **Zet stap 09 op 0** en maak de Brink spanningsloos |
| 0 V gemeten | Meet in **V DC** en controleer stap 08 en 09 |
| `unable to open …:9999` | Adapter niet bereikbaar: voeding, wifi of een gewijzigd IP-adres |
| `write message excellent FanMode not found` | Gebruik `SetFanModeAlt` in plaats van `FanMode` |

## Credits

Zie de [Engelse README](README.md#credits).

*Brink is een handelsmerk van de eigenaar. Dit project is onafhankelijk en niet verbonden aan Brink Climate Systems. Gebruik op eigen risico.*
