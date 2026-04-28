***

```markdown
# AI Energy Master v2.0 ⚡🤖

An advanced local AI orchestration system for Home Assistant. This project uses n8n and local LLMs (like Qwen2.5-Coder) to intelligently manage high-load appliances—specifically EV charging and pool heating—while protecting your main circuit breaker.

## 🌟 Key Features in v2.0

*   **🧠 Intelligent Power Budgeting:** Merges multiple high-load devices into one AI brain to prevent overloading the house.
*   **🔌 "The Zoe Fix" (KISS Principle):** Simplifies EV charging. If the cable is plugged in (`awaiting_start`), the AI understands it's time to charge. No complex SoC tracking required for non-connected cars like Renault Zoe.
*   **☀️ Solar-Aware Logic:** Automatically calculates solar surplus and prioritizes green energy.
*   **🕹️ Manual Overrides:** Integration with Home Assistant "Helpers" (Toggles) allowing the user to force-start devices. The AI automatically adjusts other loads to compensate.
*   **📝 Reasoning Logs:** The AI explains *why* it made a decision, sending the reasoning directly to a text sensor on your Home Assistant dashboard.
*   **🛡️ Safety First:** Hardcoded limits (e.g., min 8A for Zoe) to ensure charger stability and fuse protection.

---

## 🛠️ Setup Guide

### 1. Home Assistant Helpers
Create the following helpers in Home Assistant (**Settings -> Devices & Services -> Helpers**):

| Name | Type | Entity ID |
| :--- | :--- | :--- |
| **Tvinga Poolvärme** | Toggle | `input_boolean.tvinga_poolvarme` |
| **Tvinga Billaddning** | Toggle | `input_boolean.tvinga_billaddning` |
| **AI Energi Master** | Text | `input_text.ai_energi_master` |

### 2. n8n Workflow
1. Download the `.json` workflow from the `n8n-workflows/` folder in this repo.
2. Import it into your n8n instance.
3. Ensure your Home Assistant credentials are set up correctly.

### 3. The Master Prompt
Copy and paste the following prompt into your **AI Agent** node in n8n. This is the brain of the system.

```text
Du är husets Energi-Master. Ditt uppdrag är att styra både elbilsladdningen och poolvärmen smart och effektivt.
Huvudregeln är att ALDRIG överbelasta husets huvudsäkring!

DATA JUST NU:
- Husets grundförbrukning: {{ $json.state }} W
- Solproduktion: {{ $('Solceller').item.json.state }} W
- Solöverskott: {{ $('Solceller').item.json.state - $json.state }} W
- Elpris: {{ $('Elpris').item.json.state }} kr/kWh
- Kabel i bilen: {{ $('Är kablen till bilarna i?').item.json.state }}
- MANUELL KNAPP TVINGA POOL: {{ $('Tvinga Pool').item.json.state }}
- MANUELL KNAPP TVINGA BIL: {{ $('Tvinga Bil').item.json.state }}

--- MANUELL ÖVERSTYRNING ---
1. Om "MANUELL KNAPP TVINGA POOL" är "on", MÅSTE "pool_action" vara "on".
2. Om "MANUELL KNAPP TVINGA BIL" är "on" OCH kabeln är i, MÅSTE "ev_action" vara "charge" (16A).

--- REGLER FÖR ELBILEN ---
1. Om kabeln är "disconnected", sätt "ev_action" till "wait".
2. Om kabeln är i ("awaiting_start"/"connected"), ladda om priset är lågt eller solöverskott finns.
3. VIKTIGT: Sätt aldrig "ev_amps" under 8A (Zoe-krav). Prioritera bilen över poolen vid behov.

--- REGLER FÖR POOLEN ---
1. Kör poolen primärt vid solöverskott eller elpris under 0.30 kr.

SVARA ENDAST MED JSON:
{
  "ev_action": "charge/wait",
  "ev_amps": 8-16,
  "pool_action": "on/off",
  "reason": "Kort motivering."
}
```

---

## 📺 Dashboard Example
Add a Markdown card to your dashboard to see the AI's thoughts in real-time:

```yaml
type: markdown
content: >
  ## 🤖 AI Energy Status
  **Current Decision:** {{ states('input_text.ai_energi_master') }}
```

## 📄 License
MIT
```

***
