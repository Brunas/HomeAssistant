# HomeAssistant

Repository contains various home assistant scripts and automations

## AWTRIX Calendar Notifier

Sends notification about upcoming calendar event to your AWTRIX lite device. I've started to work from creating Google Calendar with all my schedules, e.g. trash emptying. Any icons can be used for this.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FBrunas%2FHomeAssistant%2Fmaster%2Fblueprints%2Fautomation%2Fawtrix_calendar_notifier.yaml)

## AWTRIX Electricity Prices

This blueprint displays real-time electricity prices on an **AWTRIX (Ulanzi TC001)** clock with high-resolution detail. Unlike standard hourly displays, this automation uses **15-minute interval data** to provide a more precise look at price fluctuations.

**Key Features:**

* **Sliding 6-Hour Window:** The graph doesn't reset at midnight; instead, the first pixel ($x=8$) always represents the **current 15-minute price**, followed by 23 pixels showing the next **5 hours and 45 minutes** of upcoming prices.
* **Today & Tomorrow Integration:** Automatically merges data from "Today" and "Tomorrow" sensors to ensure the graph remains full even late in the evening.
* **Dynamic Color Gradient:** The main text and every individual pixel in the graph are color-coded based on your custom price thresholds (e.g., Green for cheap, Red for expensive), with smooth color interpolation for the main text.
* **Precision Indexing:** Uses a mathematical offset $(Hour \times 4 + Minute \div 15)$ to ensure the display perfectly syncs with the current 15-minute billing slot.
* **Efficient MQTT Handling:** Pre-calculates the entire JSON payload as a single string to avoid Home Assistant template errors.

Following template sensors need to be defined to retrieve Nordpool prices in 15-minute intervals:
    1. Get YOUR_CONFIG_ENTRY_ID in developer tools while trying to execute `nordpool.get_prices_for_date` and turning on YAML view
    2. Change `0.15409`, `0.09601` and `1.21` to your providers daylight margin, night margin and VAT multiplier in your. Margins include VAT already.

```yaml
template:
    - trigger:
        - platform: time_pattern
        hours: "/1"
        - platform: homeassistant
        event: start
    action:
        - action: nordpool.get_prices_for_date
        data:
            config_entry: "YOUR_CONFIG_ENTRY_ID"
            date: "{{ now().date() }}"
        response_variable: today_raw
        - action: nordpool.get_prices_for_date
        data:
            config_entry: "YOUR_CONFIG_ENTRY_ID"
            date: "{{ (now() + timedelta(days=1)).date() }}"
        response_variable: tomorrow_raw
    sensor:
        - name: "Electricity Prices Today"
        unique_id: "electricity_prices_today"
        state: "{{ now().strftime('%Y-%m-%d') }}"
        attributes:
            prices: >
            {% set day_m = 0.15409 %}
            {% set night_m = 0.09601 %}
            {% set ns = namespace(items=[]) %}
            {% set raw_list = today_raw.get('LT', []) %}
            {% if raw_list | count > 0 %}
                {% for item in raw_list %}
                {% set dt = as_datetime(item.start) %}
                {% set hour = dt.hour %}
                {% set is_weekend = dt.isoweekday() >= 6 %}
                {% set base_with_vat = (item.price / 1000) * 1.21 %}
                {% if is_weekend %}
                    {% set p = base_with_vat + night_m %}
                {% else %}
                    {% if hour >= 7 and hour < 22 %}
                    {% set p = base_with_vat + day_m %}
                    {% else %}
                    {% set p = base_with_vat + night_m %}
                    {% endif %}
                {% endif %}
                {% set ns.items = ns.items + [p | round(3)] %}
                {% endfor %}
            {% endif %}
            {{ ns.items }}
        - name: "Electricity Prices Tomorrow"
        unique_id: "electricity_prices_tomorrow"
        state: "{{ (now() + timedelta(days=1)).strftime('%Y-%m-%d') }}"
        attributes:
            prices: >
            {% set day_m = 0.15409 %}
            {% set night_m = 0.09601 %}
            {% set ns = namespace(items=[]) %}
            {% set raw_list = tomorrow_raw.get('LT', []) %}

            {% if raw_list | count > 0 %}
                {% for item in raw_list %}
                {% set dt = as_datetime(item.start) %}
                {% set hour = dt.hour %}
                {% set is_weekend = dt.isoweekday() >= 6 %}
                {% set base_with_vat = (item.price / 1000) * 1.21 %}
                
                {% if is_weekend %}
                    {% set p = base_with_vat + night_m %}
                {% else %}
                    {% if hour >= 7 and hour < 22 %}
                    {% set p = base_with_vat + day_m %}
                    {% else %}
                    {% set p = base_with_vat + night_m %}
                    {% endif %}
                {% endif %}
                {% set ns.items = ns.items + [p | round(3)] %}
                {% endfor %}
            {% endif %}
            {{ ns.items }}

```

Based on an outdated Nordpool blueprint.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FBrunas%2FHomeAssistant%2Fmaster%2Fblueprints%2Fautomation%2Fawtrix_electricity_prices.yaml)