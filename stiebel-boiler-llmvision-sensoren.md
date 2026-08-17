# Stiebel Eltron SHZ 80 LCD – Display als echte HA-Sensoren (LLM Vision + Gemini)

## Architektur

Ein `template:`-Block mit drei Triggern (stündlich, HA-Start, nach SwitchBot-Tastendruck) macht einen `llmvision.image_analyzer`-Aufruf, lässt Gemini striktes JSON zurückgeben, fünf Template-Sensoren lesen direkt daraus, und ein `input_boolean` wird bei Bedarf klebrig gesetzt.

| Entity | Typ | Aktualisiert bei | Reset |
|---|---|---|---|
| `sensor.boiler_mischwassermenge` (L) | Sensor | `display_mode: standard` | automatisch |
| `binary_sensor.boiler_heizt` | Binary Sensor | `display_mode: standard` | automatisch |
| `sensor.boiler_zieltemperatur` (°C) | Sensor | `display_mode: temperature` | automatisch |
| `binary_sensor.boiler_verkalkung` | Binary Sensor | `display_mode: standard` | automatisch |
| `binary_sensor.boiler_eco_aktiv` | Binary Sensor | `display_mode: standard` | automatisch |
| `input_boolean.boiler_stoerung` | Helfer (klebrig) | wird nur AN gesetzt | **manuell** |

## Wichtige Erkenntnisse aus der Praxis

- **Modell-Verfügbarkeit:** `gemini-3.7-flash` kann bei hoher Nachfrage kurzzeitig ausgelastet sein. Aktuell auf `gemini-3.6-flash` eingestellt (gleiche Struktur, einfach zurücktauschbar).
- **Fluent vs. Clear:** Clear-Stream hat in der Praxis keine bessere Bildqualität gebracht als Fluent, trotz höherer Nennauflösung. Deshalb zurück auf `camera.reolink_boiler_fluent`.
- **`max_tokens: 2000`** ist bewusst großzügig: Gemini denkt standardmäßig nach ("Thinking"), diese Denk-Tokens zählen zum selben Budget wie die sichtbare Antwort.
- **`display_mode` als Leitplanke:** Ohne expliziten Hinweis hat Gemini bei `display_mode: "standard"` trotzdem gelegentlich eine (halluzinierte) Zieltemperatur für `target_temp_c` geliefert. Fix: eine explizite Zeile direkt nach der Liter-Beschreibung im Prompt, die klarstellt, dass die Zieltemperatur in diesem Modus NICHT angezeigt wird und -1 zurückzugeben ist.
- **"Beschreibe-erst-dann-entscheide"-Muster für alle vier Symbol-Booleans:** Für jedes der vier Symbole (Heizung, Verkalkung, Störung, ECO) gibt es ein Text-Feld direkt vor dem zugehörigen Boolean im Schema. Bei strukturierter JSON-Ausgabe generiert das Modell die Felder in Schema-Reihenfolge, muss sich also erst in Worten festlegen, was es sieht, bevor es die Bool-Entscheidung trifft – das hat beim Heizsymbol eine reale Fehlklassifizierung behoben.
- **Feste Positionen laut Anleitung (S. 6 und 8):** ECO oben rechts über "l". Heizsymbol links unten, direkt über "WW". "Ca" und das Schraubenschlüssel-Symbol unten rechts neben "WW" (andere Ecke als das Heizsymbol) – die genaue Reihenfolge der beiden zueinander ist aus der Anleitung nicht eindeutig ablesbar, aber "rechts neben WW" grenzt die Suche für das Modell gut ein.

---

## Schritt 1 – Provider einrichten

1. Falls noch nicht installiert: HACS → Integrationen → „LLM Vision" suchen → Download → Home Assistant neu starten.
2. Einstellungen → Geräte & Dienste → Integration hinzufügen → „LLM Vision".
3. Als Provider **Google Gemini** wählen, API-Key eintragen, z. B. „Google Gemini" als Name vergeben.
4. Provider-ID: Entwicklertools → Aktionen → „LLM Vision: Bild-Analysator" → Provider in der UI auswählen → auf „In YAML bearbeiten" umschalten, ID kopieren.

---

## Schritt 2 – Prompt und JSON-Schema

### JSON-Schema (`structure`)

```yaml
type: object
properties:
  display_mode:
    type: string
    enum: ["standard", "temperature", "unreadable"]
  liters:
    type: integer
  heating_symbol_observation:
    type: string
  heating_active:
    type: boolean
  scale_symbol_observation:
    type: string
  scale_warning:
    type: boolean
  fault_symbol_observation:
    type: string
  fault_warning:
    type: boolean
  eco_symbol_observation:
    type: string
  eco_symbol_on:
    type: boolean
  target_temp_c:
    type: integer
required: ["display_mode", "liters", "heating_symbol_observation", "heating_active", "scale_symbol_observation", "scale_warning", "fault_symbol_observation", "fault_warning", "eco_symbol_observation", "eco_symbol_on", "target_temp_c"]
additionalProperties: false
```

Jedes `*_observation`-Feld steht bewusst **direkt vor** seinem zugehörigen Boolean.

### Prompt

```text
Du analysierst einen Kamera-Schnappschuss vom LCD-Display eines
Warmwasserboilers (Stiebel Eltron SHZ 80 LCD).

Wichtig: Im Bild ist möglicherweise rechts neben dem Display zusätzlich ein
aufgeklebtes Infoblatt oder Foto mit einer ähnlich aussehenden
Displayabbildung zu sehen - ignoriere das vollständig und lies
ausschließlich das echte, beleuchtete LCD-Display des physischen Geräts.

Das Display zeigt normalerweise (Standardanzeige) eine 2- bis 3-stellige
Zahl gefolgt von "l" (das "l" steht direkt unter der "ECO"-Anzeige) - die
verfügbare Mischwassermenge in Litern (realistische Werte: 0-300). Steht
dort stattdessen "< 10 l", melde liters als 5. Wenn du diese
Standardanzeige siehst, wird die Zieltemperatur NICHT angezeigt, gib dann
also -1 für target_temp_c zurück.

In der Standardanzeige können an folgenden festen Positionen zusätzlich
folgende Symbole erscheinen:
- Rechts oben, über dem "l": "ECO"-Schriftzug (kann blinken, aus einem
  Einzelbild nicht von dauerhaftem Leuchten unterscheidbar).
- Links unten: Heizsymbol - eine flache gezackte Linie ist IMMER sichtbar,
  unabhängig vom Heizstatus. Nur wenn zusätzlich 3 kurze senkrechte
  Striche/Wellenlinien DIREKT ÜBER dieser Linie klar erkennbar sind, heizt
  das Gerät gerade. Ist ausschließlich die gezackte Linie zu sehen (Bereich
  darüber leer), heizt das Gerät NICHT.
- Unten rechts, neben "WW": "Ca" - erscheint bei Verkalkung des
  Heizflansches.
- Ebenfalls unten rechts, neben "WW" (nahe bei "Ca"): ein
  Schraubenschlüssel-Symbol - erscheint bei einer Störung (kann ebenfalls
  blinken, aus einem Einzelbild nicht bestimmbar).

Beschreibe bei jedem der vier Symbole zuerst in Worten, was du an der
jeweiligen Position tatsächlich siehst, bevor du dich auf true/false
festlegst.

Zeigt das Display stattdessen eine Temperatur in °C (realistische Werte:
20-85) - das passiert entweder kurzzeitig während die Plus-/Minus-Taste
gedrückt wird, ODER dauerhaft, wenn die eingestellte Soll-Temperatur unter
40 °C liegt - dann sind die obigen Zusatzsymbole nicht zuverlässig
auswertbar.

Gib ausschließlich das JSON-Objekt gemäß Schema zurück:
- display_mode: "standard" wenn die Literzahl (oder "< 10 l") zu sehen
  ist, "temperature" wenn eine Temperatur in °C zu sehen ist, "unreadable"
  wenn nichts davon eindeutig erkennbar ist (z.B. Spiegelung, Unschärfe,
  Verdeckung)
- liters: abgelesene Literzahl bei display_mode "standard" (5 falls
  "< 10 l"), sonst -1
- heating_symbol_observation: was links unten tatsächlich zu sehen ist -
  nur die gezackte Linie, oder zusätzlich Striche/Wellenlinien darüber
- heating_active: true nur wenn in heating_symbol_observation zusätzliche
  Striche über der gezackten Linie beschrieben wurden, sonst false
- scale_symbol_observation: was unten rechts neben "WW" zu "Ca" zu sehen
  ist - vorhanden oder nicht
- scale_warning: true nur wenn in scale_symbol_observation "Ca" als
  vorhanden beschrieben wurde, sonst false
- fault_symbol_observation: was unten rechts neben "WW" zum
  Schraubenschlüssel-Symbol zu sehen ist - vorhanden oder nicht
- fault_warning: true nur wenn in fault_symbol_observation das
  Schraubenschlüssel-Symbol als vorhanden beschrieben wurde, sonst false
- eco_symbol_observation: was rechts oben über "l" zu "ECO" zu sehen ist -
  vorhanden oder nicht
- eco_symbol_on: true nur wenn in eco_symbol_observation "ECO" als
  vorhanden beschrieben wurde, sonst false
- target_temp_c: abgelesene Temperatur bei display_mode "temperature",
  sonst -1
```

---

## Schritt 3 – Die vollständige Konfiguration

### Helfer anlegen

Einstellungen → Geräte & Dienste → Helfer → Hilfsobjekt hinzufügen → „Ein-/Ausschalter" → Name „Boiler Störung". Oder per YAML:

```yaml
input_boolean:
  boiler_stoerung:
    name: "Boiler Störung"
    icon: mdi:wrench-alert-outline
```

### Template-Sensoren + Automation-Logik

In `configuration.yaml` (oder `templates.yaml`):

```yaml
template:
  - trigger:
      # Stündlich
      - trigger: time_pattern
        hours: "/1"
      # Einmal beim HA-Start
      - trigger: homeassistant
        event: start
      # Sofort nach einem SwitchBot-Tastendruck (3 Sek. Puffer)
      - trigger: state
        entity_id:
          - switch.bot_85
          - switch.bot_6b
        to: "off"
        for:
          seconds: 3
    action:
      - action: llmvision.image_analyzer
        data:
          provider: "DEINE_PROVIDER_ID"    # In der UI aus der Dropdown-Liste wählen
          model: gemini-3.6-flash          # aktuell wegen Auslastung von 3.7 im Einsatz
          image_entity:
            - camera.reolink_boiler_fluent
          include_filename: false
          max_tokens: 2000    # großzügig, da Gemini standardmäßig "denkt" und diese
                              # Denk-Tokens auf dasselbe Budget angerechnet werden wie
                              # die sichtbare JSON-Antwort
          response_format: json
          structure:
            type: object
            properties:
              display_mode:
                type: string
                enum: ["standard", "temperature", "unreadable"]
              liters:
                type: integer
              heating_symbol_observation:
                type: string
              heating_active:
                type: boolean
              scale_symbol_observation:
                type: string
              scale_warning:
                type: boolean
              fault_symbol_observation:
                type: string
              fault_warning:
                type: boolean
              eco_symbol_observation:
                type: string
              eco_symbol_on:
                type: boolean
              target_temp_c:
                type: integer
            required: ["display_mode", "liters", "heating_symbol_observation", "heating_active", "scale_symbol_observation", "scale_warning", "fault_symbol_observation", "fault_warning", "eco_symbol_observation", "eco_symbol_on", "target_temp_c"]
            additionalProperties: false
          message: >
            Du analysierst einen Kamera-Schnappschuss vom LCD-Display eines
            Warmwasserboilers (Stiebel Eltron SHZ 80 LCD).

            Wichtig: Im Bild ist möglicherweise rechts neben dem Display
            zusätzlich ein aufgeklebtes Infoblatt oder Foto mit einer
            ähnlich aussehenden Displayabbildung zu sehen - ignoriere das
            vollständig und lies ausschließlich das echte, beleuchtete
            LCD-Display des physischen Geräts.

            Das Display zeigt normalerweise (Standardanzeige) eine 2- bis
            3-stellige Zahl gefolgt von "l" (das "l" steht direkt unter der
            "ECO"-Anzeige) - die verfügbare Mischwassermenge in Litern
            (realistische Werte: 0-300). Steht dort stattdessen "< 10 l",
            melde liters als 5. Wenn du diese Standardanzeige siehst, wird
            die Zieltemperatur NICHT angezeigt, gib dann also -1 für
            target_temp_c zurück.

            In der Standardanzeige können an folgenden festen Positionen
            zusätzlich folgende Symbole erscheinen:
            - Rechts oben, über dem "l": "ECO"-Schriftzug (kann blinken,
              aus einem Einzelbild nicht von dauerhaftem Leuchten
              unterscheidbar).
            - Links unten: Heizsymbol - eine flache gezackte Linie ist
              IMMER sichtbar, unabhängig vom Heizstatus. Nur wenn
              zusätzlich 3 kurze senkrechte Striche/Wellenlinien DIREKT
              ÜBER dieser Linie klar erkennbar sind, heizt das Gerät
              gerade. Ist ausschließlich die gezackte Linie zu sehen
              (Bereich darüber leer), heizt das Gerät NICHT.
            - Unten rechts, neben "WW": "Ca" - erscheint bei Verkalkung
              des Heizflansches.
            - Ebenfalls unten rechts, neben "WW" (nahe bei "Ca"): ein
              Schraubenschlüssel-Symbol - erscheint bei einer Störung
              (kann ebenfalls blinken, aus einem Einzelbild nicht
              bestimmbar).

            Beschreibe bei jedem der vier Symbole zuerst in Worten, was du
            an der jeweiligen Position tatsächlich siehst, bevor du dich
            auf true/false festlegst.

            Zeigt das Display stattdessen eine Temperatur in °C
            (realistische Werte: 20-85) - das passiert entweder kurzzeitig
            während die Plus-/Minus-Taste gedrückt wird, ODER dauerhaft,
            wenn die eingestellte Soll-Temperatur unter 40 °C liegt - dann
            sind die obigen Zusatzsymbole nicht zuverlässig auswertbar.

            Gib ausschließlich das JSON-Objekt gemäß Schema zurück:
            - display_mode: "standard" wenn die Literzahl (oder "< 10 l")
              zu sehen ist, "temperature" wenn eine Temperatur in °C zu
              sehen ist, "unreadable" wenn nichts davon eindeutig
              erkennbar ist (z.B. Spiegelung, Unschärfe, Verdeckung)
            - liters: abgelesene Literzahl bei display_mode "standard" (5
              falls "< 10 l"), sonst -1
            - heating_symbol_observation: was links unten tatsächlich zu
              sehen ist - nur die gezackte Linie, oder zusätzlich
              Striche/Wellenlinien darüber
            - heating_active: true nur wenn in heating_symbol_observation
              zusätzliche Striche über der gezackten Linie beschrieben
              wurden, sonst false
            - scale_symbol_observation: was unten rechts neben "WW" zu
              "Ca" zu sehen ist - vorhanden oder nicht
            - scale_warning: true nur wenn in scale_symbol_observation
              "Ca" als vorhanden beschrieben wurde, sonst false
            - fault_symbol_observation: was unten rechts neben "WW" zum
              Schraubenschlüssel-Symbol zu sehen ist - vorhanden oder
              nicht
            - fault_warning: true nur wenn in fault_symbol_observation das
              Schraubenschlüssel-Symbol als vorhanden beschrieben wurde,
              sonst false
            - eco_symbol_observation: was rechts oben über "l" zu "ECO" zu
              sehen ist - vorhanden oder nicht
            - eco_symbol_on: true nur wenn in eco_symbol_observation "ECO"
              als vorhanden beschrieben wurde, sonst false
            - target_temp_c: abgelesene Temperatur bei display_mode
              "temperature", sonst -1
        response_variable: boiler_vision

      # Klebriges Störungs-Flag: wird nur AN gesetzt, nie automatisch AUS
      - if:
          - condition: template
            value_template: >
              {% set raw = (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim %}
              {% set data = (raw | from_json) if raw.startswith('{') else None %}
              {{ data is not none and data.display_mode == 'standard' and data.fault_warning }}
        then:
          - action: input_boolean.turn_on
            target:
              entity_id: input_boolean.boiler_stoerung

    sensor:
      - name: "Boiler Mischwassermenge"
        unique_id: boiler_mischwassermenge
        unit_of_measurement: "L"
        state_class: measurement
        icon: mdi:water-boiler
        variables:
          raw: "{{ (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim }}"
          data: "{{ (raw | from_json) if raw.startswith('{') else None }}"
        state: >
          {{ (data.liters if (data and data.display_mode == 'standard' and 0 <= data.liters <= 300) else this.state) }}

      - name: "Boiler Zieltemperatur"
        unique_id: boiler_zieltemperatur
        unit_of_measurement: "°C"
        device_class: temperature
        state_class: measurement
        variables:
          raw: "{{ (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim }}"
          data: "{{ (raw | from_json) if raw.startswith('{') else None }}"
        state: >
          {{ (data.target_temp_c if (data and data.display_mode == 'temperature' and 20 <= data.target_temp_c <= 85) else this.state) }}

    binary_sensor:
      - name: "Boiler heizt"
        unique_id: boiler_heizt
        device_class: running
        variables:
          raw: "{{ (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim }}"
          data: "{{ (raw | from_json) if raw.startswith('{') else None }}"
        state: >
          {{ (data.heating_active if (data and data.display_mode == 'standard') else (this.state == 'on')) }}
        attributes:
          beobachtung: >
            {{ (data.heating_symbol_observation if (data and data.display_mode == 'standard') else this.attributes.get('beobachtung', '(noch keine Daten)')) }}

      - name: "Boiler Verkalkung"
        unique_id: boiler_verkalkung
        device_class: problem
        icon: mdi:water-alert-outline
        variables:
          raw: "{{ (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim }}"
          data: "{{ (raw | from_json) if raw.startswith('{') else None }}"
        state: >
          {{ (data.scale_warning if (data and data.display_mode == 'standard') else (this.state == 'on')) }}
        attributes:
          beobachtung: >
            {{ (data.scale_symbol_observation if (data and data.display_mode == 'standard') else this.attributes.get('beobachtung', '(noch keine Daten)')) }}

      - name: "Boiler ECO aktiv"
        unique_id: boiler_eco_aktiv
        icon: mdi:leaf
        variables:
          raw: "{{ (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim }}"
          data: "{{ (raw | from_json) if raw.startswith('{') else None }}"
        state: >
          {{ (data.eco_symbol_on if (data and data.display_mode == 'standard') else (this.state == 'on')) }}
        attributes:
          beobachtung: >
            {{ (data.eco_symbol_observation if (data and data.display_mode == 'standard') else this.attributes.get('beobachtung', '(noch keine Daten)')) }}
```

Danach: Entwicklertools → YAML → Konfiguration prüfen, dann Template-Entities + Input-Booleans neu laden oder Home Assistant neu starten.

**Manueller Reset der Störung:** Einstellungen → Geräte & Dienste → Helfer → „Boiler Störung" öffnen und den Schalter antippen.

**Debugging über die `beobachtung`-Attribute:** Bei `boiler_heizt`, `boiler_verkalkung` und `boiler_eco_aktiv` siehst du in Entwicklertools → Zustände jeweils, was das Modell beim letzten Standard-Snapshot an der jeweiligen Position tatsächlich beschrieben hat. Für `input_boolean.boiler_stoerung` geht das nicht (Helfer haben keine eigenen Attribute) – falls du das auch für die Störung brauchst, sag Bescheid, dann ergänze ich einen separaten Text-Sensor dafür.

---

## Testen

Zwei getrennte Dinge, die schiefgehen können: die LLM-Antwort selbst (liest Gemini das Display richtig?) und die Jinja-Logik, die daraus die Sensorwerte baut (wird das JSON richtig geparst und den richtigen Feldern zugeordnet?). Beides lässt sich unabhängig voneinander prüfen, bevor du der ganzen Kette vertraust.

### 1. LLM-Antwort direkt prüfen (ohne Sensoren anzufassen)

Entwicklertools → Aktionen → „LLM Vision: Bild-Analysator" (`llmvision.image_analyzer`) auswählen. Trage exakt dieselben Felder wie in der Automation ein (Provider, Modell `gemini-3.6-flash`, `image_entity: camera.reolink_boiler_fluent`, den Prompt aus Schritt 2, `response_format: json`, das Schema aus Schritt 2, `max_tokens: 2000`) und führe aus. Im Ergebnis siehst du `response_text` – das ist die rohe JSON-Antwort. Prüfe:
- Ist es valides JSON (alle 11 Felder da, keine Markdown-Codefences drumherum)?
- Stimmen die Werte mit dem, was gerade tatsächlich am Display zu sehen ist, überein?
- Passen die vier `*_observation`-Felder zur Realität?

**Falls `response_text` mitten im Satz abbricht** (z. B. nur „Here is the"): `max_tokens` zu niedrig – hochsetzen.

Das testet nur Kamera + Gemini + Prompt, komplett unabhängig von den Template-Sensoren.

### 2. Parsing-Logik isoliert testen

Kopiere den `response_text`-Wert aus Schritt 1 und füge folgendes in Entwicklertools → Vorlage ein:

```jinja
{% set boiler_vision = {'response_text': 'HIER DEIN response_text EINFÜGEN'} %}
{% set raw = (boiler_vision.response_text | default('')) | replace('```json','') | replace('```','') | trim %}
{% set data = (raw | from_json) if raw.startswith('{') else None %}

display_mode: {{ data.display_mode if data else 'PARSE-FEHLER' }}
liters: {{ data.liters if (data and data.display_mode == 'standard' and 0 <= data.liters <= 300) else '(Fallback)' }}
heating_active: {{ data.heating_active if (data and data.display_mode == 'standard') else '(Fallback)' }} -- Beobachtung: {{ data.heating_symbol_observation if data else '' }}
scale_warning: {{ data.scale_warning if (data and data.display_mode == 'standard') else '(Fallback)' }} -- Beobachtung: {{ data.scale_symbol_observation if data else '' }}
fault_warning: {{ data.fault_warning if (data and data.display_mode == 'standard') else '(Fallback)' }} -- Beobachtung: {{ data.fault_symbol_observation if data else '' }}
eco_symbol_on: {{ data.eco_symbol_on if (data and data.display_mode == 'standard') else '(Fallback)' }} -- Beobachtung: {{ data.eco_symbol_observation if data else '' }}
target_temp_c: {{ data.target_temp_c if (data and data.display_mode == 'temperature' and 20 <= data.target_temp_c <= 85) else '(Fallback)' }}
```

Hinweis: `this.state`/`this.attributes` gibt es im Vorlagen-Tester nicht (das existiert nur im Kontext eines echten, bereits existierenden Sensors), daher oben durch Klartext ersetzt.

### 3. Live-Test mit den echten Entitäten

Konfiguration einfügen, Template-Entities + Input-Booleans neu laden (oder Home Assistant neu starten). Statt bis zu einer Stunde auf den ersten regulären Trigger zu warten: stell testweise `hours: "/1"` auf `minutes: "/2"`, lade neu, beobachte in Entwicklertools → Zustände die Entities `sensor.boiler_mischwassermenge` etc. und deren „zuletzt geändert". Danach wieder zurück auf `hours: "/1"`.

**Wichtig:** Zum Testen NICHT `switch.bot_85` / `switch.bot_6b` manuell umschalten – das sind die echten SwitchBots, ein Umschalten drückt tatsächlich die Taste am Boiler und verstellt die Temperatur.

### 4. Bei Problemen: Log prüfen

Falls ein Sensor dauerhaft auf `unknown` steht: Einstellungen → System → Protokolle, nach „boiler" oder „TemplateError" suchen. Meistens liegt es dann an der Parsing-Logik (Schritt 2 oben hilft, das einzugrenzen) oder daran, dass `response_format: json` bei dir nicht wie erwartet greift.

---

## Push-Benachrichtigung bei Störung

```yaml
automation:
  - alias: "Boiler Störung Benachrichtigung"
    description: "Push-Nachricht, sobald das Service/Fehler-Symbol am Boiler-Display erkannt wird"
    triggers:
      - trigger: state
        entity_id: input_boolean.boiler_stoerung
        to: "on"
    actions:
      - action: notify.notify   # ANPASSEN: dein echtes Notify-Ziel, z.B. notify.mobile_app_dein_handy
        data:
          title: "⚠️ Boiler Störung"
          message: >
            Das Service/Fehler-Symbol ist am Boiler-Display aufgetaucht.
            Bitte prüfen (Kapitel 6/15 der Anleitung). Zum Zurücksetzen:
            Helfer "Boiler Störung" in Home Assistant manuell umschalten.
```

`notify.notify` ist der generische Platzhalter – ersetze ihn durch dein konkretes Ziel. Findest du unter Entwicklertools → Aktionen, Suche nach „notify" (typischerweise `notify.mobile_app_<dein-gerätename>`).

---

## Praxistipps

**Kosten:** Da Denk-Tokens zum Output-Preis mitgezählt werden und jetzt vier zusätzliche Text-Felder generiert werden, dürfte es bei 1×/Stunde eher im Bereich von 1-2 $/Monat liegen (grobe Schätzung). Nach ein paar Tagen lohnt sich ein Blick in die Google AI Studio Nutzungsübersicht.

**Beleuchtung:** Achte auf Überbelichtung durch zu direktes/nahes Licht.

**SwitchBot-Platzierung:** Falls ein Symbol weiterhin auffällig danebenliegt, lohnt sich ein Blick, ob ein SwitchBot die jeweilige Position physisch verdeckt.

---

## Erweiterungsideen

- **Service-Code, Energieverbrauch (kWh), Auslauftemperatur:** weiterhin nur über die Menü-Taste (▶) erreichbar, bräuchte einen dritten SwitchBot.
- **Separater Beobachtungs-Sensor für die Störung:** da `input_boolean` keine Attribute unterstützt, ließe sich `fault_symbol_observation` bei Bedarf in einen eigenen `sensor.boiler_stoerung_beobachtung` schreiben.
