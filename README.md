# ⚡🤖 AI-Driven EV Charging Optimization

Smart elbilsladdning med Home Assistant, n8n och AI – helt self-hosted.

![Status](https://img.shields.io/badge/Status-Observer_Mode-blue)
![HA](https://img.shields.io/badge/Home_Assistant-2024.x+-41BDF5?logo=homeassistant)
![n8n](https://img.shields.io/badge/n8n-Self_Hosted-FF6D5A?logo=n8n)
![License](https://img.shields.io/badge/License-MIT-green)

## Vad är detta?

Ett system som optimerar elbilsladdning automatiskt genom att analysera:

- **Elpris** (Tibber Pulse kvartspris / Nordpool timpris) – laddar vid billigaste tidpunkterna
- **Prisranking** – jämför aktuellt pris mot dagens min/max/snitt/peak/off-peak
- **Solproduktion** (SolarEdge) – utnyttjar egenproducerad el
- **Bilens batterinivå** (SoC) – laddar bara vid behov
- **Fasbelastning** – skyddar huvudsäkringen

En AI-modell (Mistral) utvärderar all data var 15:e minut (kvartspris) eller varje timme (timpris) och ger ett strukturerat laddningsbeslut.

### Två workflow-varianter

| Variant | Priskälla | Intervall | Bäst för |
|---------|-----------|-----------|----------|
| **Tibber Pulse** | Kvartspris (realtid) | Var 15:e minut | Tibber-kunder med Pulse |
| **Nordpool** | Timpris | Varje timme | Alla med Nordpool-integration |

## Arkitektur

```
┌─────────────────────────────────────────────┐
│  Lager 1: HA-automation (SÄKERHETSNÄT)      │
│  • Alltid aktivt                            │
│  • ON/OFF-styrning av laddbox               │
│  • Deadline, nödladdning, lastbalansering   │
└──────────────────┬──────────────────────────┘
                   │ data var 15:e min (Tibber) / varje timme (Nordpool)
┌──────────────────▼──────────────────────────┐
│  Lager 2: n8n + AI (OPTIMERING)             │
│  • Analyserar elpris + SoC + sol            │
│  • Mistral AI ger laddningsbeslut (JSON)    │
│  • Notifierar i HA (observer-läge)          │
│  • Framtid: dynamisk ampere-styrning        │
└─────────────────────────────────────────────┘
```

> **Nyckelprincip:** Om AI:n eller n8n ligger nere fungerar Lager 1 som innan. Bilen laddas alltid vid behov.

## Hårdvarukrav

| Komponent | Testad med | Alternativ |
|-----------|-----------|------------|
| Laddbox | Easee A1 | Zaptec, Wallbox, Go-e |
| Elbil | VW ID (MEB) | Tesla, Volvo, Hyundai/Kia |
| Solpaneler | SolarEdge | Huawei, Fronius, Enphase |
| Elpris | Tibber Pulse (kvartspris) | Nordpool (timpris), Entso-e |
| Elmätare | Tibber Pulse (P1) | HomeWizard, Slimmelezer |
| Server | Proxmox (LXC) | Docker, Raspberry Pi |

## Snabbstart

### 1. Installera n8n på Proxmox

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/n8n.sh)"
```

Rekommenderade resurser: 2 GB RAM, 2 CPU-kärnor, 8 GB disk.

### 2. Skapa HA Access Token

Home Assistant → Profil → Security → Long-Lived Access Tokens → Create Token

### 3. Importera n8n-workflow

1. Öppna n8n (`http://<IP>:5678`)
2. Importera **en** av:
   - `n8n-workflows/EV Smart Charging - Tibber Pulse Quarter Price.json` (kvartspris, rekommenderat)
   - `n8n-workflows/EV Smart Charging - AI Observer.json` (Nordpool timpris)
3. Uppdatera credentials (HA-token, Mistral API-nyckel, Google Sheets service account)
4. Uppdatera entity-IDs (se [Anpassning](#anpassning))

### 4. Importera HA-automationer

Kopiera filerna i `ha-automations/` till din HA `automations.yaml` eller importera via HA UI.

### 5. Aktivera

Publish workflowet i n8n → det körs var 15:e minut (Tibber) eller varje timme (Nordpool) i observer-läge.

## Anpassning

### Entity-ID Mappning

Redigera filen `templates/entity-config.yaml` med dina entiteter:

```yaml
# Laddbox
charger_status: sensor.a1_status              # Ändra till din laddbox
charger_switch: switch.a1_laddboxen_aktiverad
charger_power: sensor.a1_effekt

# Bil
car_soc: sensor.wv2zzzeb9rh001501_battery_level  # Ändra till din bil
car_range: sensor.wv2zzzeb9rh001501_electric_range
car_cable: binary_sensor.wv2zzzeb9rh001501_charging_cable_connected

# Energi
electricity_price: sensor.nordpool_kwh_se3_sek_2_10_025
solar_production: sensor.solaredge_aktuell_effekt
cheap_electricity: binary_sensor.billig_el

# Lastbalansering
phase_load: sensor.huset_max_fasbelastning
pool_heater: switch.poolvarme_ovrigt_2
```

### Tröskelvärden

| Parameter | Default | Beskrivning |
|-----------|---------|-------------|
| `cheap_price_threshold` | 1.00 kr/kWh | Ladda under detta pris |
| `medium_price_threshold` | 2.00 kr/kWh | Ladda bara med sol/låg SoC |
| `emergency_soc` | 15% | Nödladda oavsett pris |
| `fuse_limit_high` | 24.5A | Stäng av laddning |
| `fuse_limit_low` | 20.0A | Återställ (hysteres) |
| `fuse_delay_off` | 45 sek | Vänta ut startpikar |
| `fuse_delay_on` | 5 min | Vänta före återställning |
| `solar_min_watts` | 2000W | Minimum för sol-laddning |

## Filstruktur

```
ai-ev-charging/
├── README.md                          # Denna fil
├── LICENSE
├── docs/
│   ├── systemdokumentation.html       # Komplett systemdokumentation
│   └── analysrapport-feb2026.html     # Första analysrapporten (66 tim data)
├── n8n-workflows/
│   ├── EV Smart Charging - AI Observer.json          # Nordpool (timpris)
│   └── EV Smart Charging - Tibber Pulse Quarter Price.json  # Tibber (kvartspris)
├── ha-automations/
│   ├── smart-charging.yaml            # Huvudautomation: smart laddning
│   ├── load-protection-off.yaml       # Säkerhet: rädda huvudsäkringen
│   └── load-protection-restore.yaml   # Säkerhet: återställ efter hög last
└── templates/
    └── entity-config.yaml             # Mall för entity-IDs
```

## AI-beslut

### Tibber Pulse (kvartspris) – rekommenderat

AI:n använder Tibbers `intraday_price_ranking` (0% = billigast, 100% = dyrast) för smartare beslut:

| Prisranking | Beslut | Ampere |
|-------------|--------|--------|
| < 25% (billigaste kvarten) | ✅ Ladda alltid | 16A |
| Under genomsnitt | ✅ Ladda | 12–16A |
| > 75% (dyraste kvarten) | ⚠️ Vänta om SoC > 30% | 0A |
| Över peak-nivå | ❌ Vänta om möjligt | 0A |
| SoC < 30% | 🔴 Ladda oavsett pris | 16A |

### Nordpool (timpris) – alternativ

| Elpris | Beslut | Ampere |
|--------|--------|--------|
| < 1 kr/kWh | ✅ Ladda | 16A (max) |
| Under dagens genomsnitt | ✅ Ladda | 12–16A |
| Nära dagens lägsta (inom 10 öre) | ✅ Ladda | 16A |
| 1–2 kr/kWh (över snitt) | ⚠️ Bara med sol eller låg SoC | 6–12A |
| > 2 kr/kWh | ❌ Vänta | 0A |

> **Lärdom:** Statiska trösklar ("ladda under 1 kr") missar att 1.08 kr kan vara nattens billigaste pris. AI:n jämför därför mot dagens min/max/genomsnitt för smartare beslut.

### Exempel (Tibber Pulse)

```json
{
  "action": "charge",
  "reason": "Prisranking 12% – bland billigaste kvarten idag",
  "recommended_amps": 16
}
```

## Roadmap

- [x] **Fas 1:** Observer-läge – AI loggar beslut utan styrning
- [x] **Fas 1b:** Google Sheets-loggning – AI vs HA jämförelse
- [x] **Fas 1c:** Kvartsprisoptimering med Tibber Pulse (var 15:e min)
- [x] **Fas 1d:** [Första analysrapporten](docs/analysrapport-feb2026.html) – 66 timmar data
- [ ] **Fas 2:** Dynamisk ampere-styrning (6–16A baserat på sol/pris)
- [ ] **Fas 3:** Mönsterigenkänning (körmönster, veckodagar, väder)
- [ ] **Fas 4:** Helhemoptimering (belysning, uppvärmning, effekttoppar)

## Besparingsanalys (Google Sheets)

Systemet loggar automatiskt varje kvarts data till Google Sheets för att jämföra AI:ns rekommendationer mot HA-automationens faktiska beslut.

**Loggade datapunkter:**

| Kolumn | Beskrivning |
|--------|-------------|
| Timestamp | Tidpunkt för analys |
| Elpris | Aktuellt pris (kr/kWh) |
| Pris_Min | Dagens lägsta pris |
| Pris_Max | Dagens högsta pris |
| Pris_Snitt | Dagens genomsnittspris |
| SoC | Bilens batterinivå (%) |
| Sol_W | Solproduktion (W) |
| AI_Action | AI:ns rekommendation (charge/wait) |
| AI_Amps | Rekommenderad laddström (0–16A) |
| HA_Laddade | Laddboxens faktiska status |

**Setup:**
1. Skapa ett Google Sheet med headers enligt ovan
2. Skapa en Service Account i Google Cloud Console
3. Aktivera Google Sheets API + Google Drive API
4. Dela sheetet med service accountens e-postadress
5. Lägg till Google Sheets-noden i n8n-workflowet

Datan kan sedan användas för att beräkna: *"Om AI:n hade styrt – hur mycket hade vi sparat jämfört med den vanliga automationen?"*

## Två-bilars-hantering

Systemet hanterar två elbilar som delar en laddbox:

- **Uppkopplad bil** (t.ex. VW ID): SoC läses direkt via API
- **Ej uppkopplad bil** (t.ex. Renault Zoe): SoC anges manuellt via `input_number` i HA
- **Identifiering:** Om VW:ns kabel inte är ansluten och laddboxen visar "awaiting_start" → det är Zoe

## Bidra

Pull requests välkomna! Särskilt intresserad av:
- Stöd för fler laddboxar (Zaptec, Wallbox, Go-e)
- Fler bil-integrationer
- Förbättrade AI-prompts
- Dashboard-mallar för HA

## Tack till

- [Home Assistant](https://www.home-assistant.io/) – Smart home-plattformen
- [n8n](https://n8n.io/) – Workflow-automation
- [Mistral AI](https://mistral.ai/) – Europeisk AI-modell
- [Easee](https://easee.com/) – Laddbox-integration
- [Tibber](https://tibber.com/) – Kvartspris via Pulse
- [Nordpool](https://github.com/custom-components/nordpool) – Elpris-integration
- [Proxmox VE Helper Scripts](https://community-scripts.github.io/ProxmoxVE/) – n8n LXC-installation

## Licens

MIT – Använd fritt, dela gärna!
