AI-Driven EV Charging & Pool Optimization (v2.0)
Denna uppdatering av repot (tidigare baserat på Mistral) är nu helt fokuserad på lokala LLM-modeller (Qwen2.5-Coder) för att agera som "Energy Master". Den hanterar dynamiskt elbilsladdning och poolvärme baserat på elpris, solöverskott och husets huvudsäkring.
🚀 Nyheter i Version 2.0 (The Energy Master Update)
Samlad AI-Hjärna: Bilen och poolen styrs nu av en och samma AI-nod för att förhindra att huvudsäkringen går (Total Power Budgeting).
KISS-Principen för Elbilar ("The Zoe Fix"): Inget mer krångel med att hämta SoC%. Om kabeln är i (`awaiting_start`), antar systemet att bilen ska ha ström. Bilen måste också få minst 8A för att ladda, annars pausas den automatiskt.
Boss Mode (Manuell Överstyrning): Fysiska Home Assistant-knappar kan tvinga systemet att starta poolen eller bilen. AI:n justerar då den andra enheten för att rädda säkringen.
Smart Loggning: AI:ns beslutsprocess skickas direkt till en Home Assistant-hjälpare (`input_text`) för realtidsloggning på din dashboard.
🛠 Förutsättningar
System
Beskrivning
Home Assistant
Ditt primära smarta hem-nav. Behöver sensorer för Totalförbrukning, Solproduktion, Elpris och Bilstauts.
n8n
Automationsmotorn. Hämtar in data från HA, frågar AI:n och pushar tillbaka resultatet.
Ollama
Körs lokalt (t.ex. med Qwen2.5-Coder 7b) och är inkopplad i n8n via LangChain-noder.

📝 The Master Prompt (För din n8n AI Agent)
Kopiera detta och klistra in i din Chat Model Prompt i n8n.
Du är husets Energi-Master. Ditt uppdrag är att styra både elbilsladdningen och poolvärmen smart och effektivt.
Huvudregeln är att ALDRIG överbelasta husets huvudsäkring!

DATA JUST NU:
- Husets grundförbrukning: {{ $('Get a state1').item.json.state }} W
- Solproduktion: {{ $('Solceller').item.json.state }} W
- Solöverskott (Sol minus Förbrukning): {{ $('Solceller').item.json.state - $('Get a state1').item.json.state }} W
- Elpris: {{ $('Elpris').item.json.state }} kr/kWh
- Kabel i bilen: {{ $('Är kablen till bilarna i?').item.json.state }}
- MANUELL KNAPP TVINGA POOL: {{ $('Tvinga Pool').item.json.state }}
- MANUELL KNAPP TVINGA BIL: {{ $('Tvinga Bil').item.json.state }}

--- MANUELL ÖVERSTYRNING (CHEFENS ORDER) ---
1. Om "MANUELL KNAPP TVINGA POOL" är "on", MÅSTE "pool_action" vara "on". Pris och sol ignoreras.
2. Om "MANUELL KNAPP TVINGA BIL" är "on" OCH "Kabel i bilen" inte är disconnected, MÅSTE "ev_action" vara "charge" och "ev_amps" sättas högt (ex 16A). Pris och sol ignoreras.

--- REGLER FÖR ELBILEN (Om ej manuellt överstyrd) ---
1. KRITISK KONTROLL: Om "Kabel i bilen" är "disconnected", en tom sträng eller okänd, MÅSTE "ev_action" sättas till "wait" och "ev_amps" till 0.
2. SMART LADDNING (Ignorera batteriprocent): Om "Kabel i bilen" visar "awaiting_start", "connected", eller "charging", betyder det att bilen vill ha ström. 
- Sätt "ev_action" till "charge" OM elpriset är lågt/rimligt ELLER om det finns solöverskott.
- Sätt "ev_action" till "wait" om elpriset är högt och inget solöverskott finns (spara laddningen till natten).
3. ZOE-STRÖMKRAV (VIKTIGT): När du beslutar att ladda ("charge"), kräver bilen MINST 8A. Om husets totala budget är för tight för att ge minst 8A, MÅSTE du sätta "ev_action" till "wait". Bilen har dock prioritet över poolen, så stäng av poolen först om det krävs för att ge bilen minst 8A.

--- REGLER FÖR POOLEN (Om ej manuellt överstyrd) ---
1. Poolen är en lyx/värmebuffert. 
2. Sätt "pool_action" till "on" primärt om Solöverskottet är större än 0 W, ELLER om elpriset är under 0.30 kr. I övriga fall, sätt till "off".

--- ÖVERGRIPANDE SÄKERHET ---
1. Om husets totala förbrukning är kritiskt hög i förhållande till huvudsäkringen, och ingen manuell överstyrning tvingar driften: sänk först bilens Amps (ev_amps) och stäng sedan av poolen helt ("pool_action": "off").

SVARA ENDAST MED ETT JSON-OBJEKT I EXAKT DETTA FORMAT (inga backticks, ingen markdown):
{
  "ev_action": "charge" eller "wait",
  "ev_amps": siffran 8 till 16 (eller 0 om wait),
  "pool_action": "on" eller "off",
  "reason": "Kort motivering till varför."
}


⚙️ Home Assistant Hjälpare (Helpers)
För att bygga den fulla upplevelsen, lägg till dessa under Inställningar -> Hjälpare:
input_boolean.tvinga_poolvarme - Switch för manuell styrning.
input_boolean.tvinga_billaddning - Switch för manuell styrning av EV.
input_text.ai_energi_master - Textfält som AI:n skriver sin status till.

N8N JSON Import: Kom ihåg att ladda ner ditt flöde från n8n som en `.json`-fil och lägga i `n8n-workflows/`-mappen i ditt repo, så att besökare enkelt kan importera hela grafen.
