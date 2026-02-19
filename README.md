# ⚡🤖 AI-Driven EV Charging Optimization

Smart elbilsladdning med Home Assistant, n8n och AI – helt self-hosted.

![Status](https://img.shields.io/badge/Status-Observer_Mode-blue)
![HA](https://img.shields.io/badge/Home_Assistant-2024.x+-41BDF5?logo=homeassistant)
![n8n](https://img.shields.io/badge/n8n-Self_Hosted-FF6D5A?logo=n8n)
![License](https://img.shields.io/badge/License-MIT-green)

## Vad är detta?

Ett system som optimerar elbilsladdning automatiskt genom att analysera:

- **Elpris** (Nordpool) – laddar vid billigaste timmarna
- **Solproduktion** (SolarEdge) – utnyttjar egenproducerad el
- **Bilens batterinivå** (SoC) – laddar bara vid behov
- **Fasbelastning** – skyddar huvudsäkringen

En AI-modell (Mistral) utvärderar all data varje timme och ger ett strukturerat laddningsbeslut.

## Arkitektur

```
┌─────────────────────────────────────────────┐
│  Lager 1: HA-automation (SÄKERHETSNÄT)      │
│  • Alltid aktivt                            │
│  • ON/OFF-styrning av laddbox               │
│  • Deadline, nödladdning, lastbalansering   │
└──────────────────┬──────────────────────────┘
                   │ data varje timme
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
| Elpris | Nordpool SE3 | Tibber, Entso-e |
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
2. Importera `n8n-workflows/ev-smart-charging-observer.json`
3. Uppdatera credentials (HA-token, Mistral API-nyckel)
4. Uppdatera entity-IDs (se [Anpassning](#anpassning))

### 4. Importera HA-automationer

Kopiera filerna i `ha-automations/` till din HA `automations.yaml` eller importera via HA UI.

### 5. Aktivera

Publish workflowet i n8n → det körs varje timme i observer-läge.

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
│   └── systemdokumentation.html       # Komplett systemdokumentation
├── n8n-workflows/
│   └── ev-smart-charging-observer.json # n8n workflow (importerbar)
├── ha-automations/
│   ├── smart-charging.yaml            # Huvudautomation: smart laddning
│   ├── load-protection-off.yaml       # Säkerhet: rädda huvudsäkringen
│   └── load-protection-restore.yaml   # Säkerhet: återställ efter hög last
└── templates/
    └── entity-config.yaml             # Mall för entity-IDs
```

## AI-beslut (exempel)

```json
{
  "action": "wait",
  "reason": "Elpriset är över 1 kr/kWh och bilens batterinivå är över 30%. Vänta tills priset sjunker.",
  "recommended_amps": 0
}
```

| Elpris | Beslut | Ampere |
|--------|--------|--------|
| < 1 kr/kWh | ✅ Ladda | 16A (max) |
| 1–2 kr/kWh | ⚠️ Bara med sol eller låg SoC | 6–12A |
| > 2 kr/kWh | ❌ Vänta | 0A |

## Roadmap

- [x] **Fas 1:** Observer-läge – AI loggar beslut utan styrning
- [ ] **Fas 2:** Dynamisk ampere-styrning (6–16A baserat på sol/pris)
- [ ] **Fas 3:** Mönsterigenkänning (körmönster, veckodagar, väder)
- [ ] **Fas 4:** Helhemoptimering (belysning, uppvärmning, effekttoppar)

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
- [Nordpool](https://github.com/custom-components/nordpool) – Elpris-integration
- [Proxmox VE Helper Scripts](https://community-scripts.github.io/ProxmoxVE/) – n8n LXC-installation

## Licens

MIT – Använd fritt, dela gärna!
