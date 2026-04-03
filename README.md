# HomeAssistant

Repository contains various home assistant scripts and automations

## AWTRIX Calendar Notifier

Sends notification about upcoming calendar event to your AWTRIX lite device. I've started to work from creating Google Calendar with all my schedules, e.g. trash emptying. Any icons can be used for this.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FBrunas%2FHomeAssistant%2Fmaster%2Fblueprints%2Fautomation%2Fawtrix_calendar_notifier.yaml)

## AWTRIX Electricity Prices

This blueprint displays real-time electricity prices on an AWTRIX lite clock with high-resolution detail. Unlike standard hourly displays, this automation uses **15-minute interval data** to provide a more precise look at price fluctuations.

**Key Features:**

* **Sliding 6-Hour Window:** The graph doesn't reset at midnight; instead, the first pixel ($x=8$) always represents the **current 15-minute price**, followed by 23 pixels showing the next **5 hours and 45 minutes** of upcoming prices.
* **Today & Tomorrow Integration:** Automatically merges data from "Today" and "Tomorrow" sensors to ensure the graph remains full even late in the evening.
* **Dynamic Color Gradient:** The main text and every individual pixel in the graph are color-coded based on your custom price thresholds (e.g., Green for cheap, Red for expensive), with smooth color interpolation for the main text.
* **Precision Indexing:** Uses a mathematical offset $(Hour \times 4 + Minute \div 15)$ to ensure the display perfectly syncs with the current 15-minute billing slot.
* **Efficient MQTT Handling:** Pre-calculates the entire JSON payload as a single string to avoid Home Assistant template errors.

### Jinja2 macro

Following jinja macro is required to keep electricity price calculation in one place:

1. Change `0.15409`, `0.09601` and `1.21` to your providers daylight margin, night margin and VAT multiplier respectively.

2. In Lithuania night tariff in Summer is from midnight to 8:00, when in Winter it's from 23:00 to 7:00 - change that if it's different for you.

3. Place it in `/config/custom_templates/electricity.jinja`.

>**NOTE**: Margins include VAT already.


```jinja
{# /config/custom_templates/electricity.jinja #}

{% macro calculate_electricity_price(raw_price, dt) %}
  {# 1. Setup Constants #}
  {% set vat = 1.21 %}
  {% set day_m = 0.15409 %}
  {% set night_m = 0.09601 %}

  {# 2. Convert raw MWh to kWh and add VAT #}
  {% set price_with_vat = (raw_price / 1000) * vat %}

  {# 3. Determine Time Context from the provided datetime #}
  {% set local_dt = dt | as_local %}
  {% set hour = local_dt.hour %}
  {% set is_weekend = local_dt.isoweekday() >= 6 %}
  {% set is_summer = local_dt.timetuple().tm_isdst > 0 %}

  {# 4. Apply Tariff Logic #}
  {% if is_weekend %}
    {{ (price_with_vat + night_m) | round(3) }}
  {% else %}
    {% if is_summer %}
      {# Summer: Night 00:00 - 08:00 #}
      {{ (price_with_vat + (night_m if hour < 8 else day_m)) | round(3) }}
    {% else %}
      {# Winter: Day 07:00 - 23:00 #}
      {{ (price_with_vat + (day_m if (hour >= 7 and hour < 23) else night_m)) | round(3) }}
    {% endif %}
  {% endif %}
{% endmacro %}
```

### Template sensors

Following template sensors need to be defined to retrieve Nordpool prices in 15-minute intervals with suplemental Current 15-minute Price and Current Hourly Price sensors at the bottom:

1. Get YOUR_CONFIG_ENTRY_ID in developer tools while trying to execute `nordpool.get_prices_for_date` and turning on YAML view

2. `sensor.nord_pool_lt_dabartine_kaina` change to your NordPool integration current price sensor - it's returning hourly price.

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
                {% from 'electricity.jinja' import calculate_electricity_price %}
                {% set ns = namespace(items=[]) %}
                {% set raw_list = today_raw.get('LT', []) %}

                {% if raw_list | count > 0 %}
                    {% for item in raw_list %}
                    {# We pass the raw price and the start time to the macro #}
                    {% set final_p = calculate_electricity_price(item.price, as_datetime(item.start)) %}
                    {% set ns.items = ns.items + [final_p | float] %}
                    {% endfor %}
                {% endif %}
                {{ ns.items }}
        - name: "Electricity Prices Tomorrow"
        unique_id: "electricity_prices_tomorrow"
        state: "{{ (now() + timedelta(days=1)).strftime('%Y-%m-%d') }}"
        attributes:
            prices: >
                {% from 'electricity.jinja' import calculate_electricity_price %}
                {% set ns = namespace(items=[]) %}
                {% set raw_list = tomorrow_raw.get('LT', []) %}

                {% if raw_list | count > 0 %}
                    {% for item in raw_list %}
                    {# We pass the raw price and the start time to the macro #}
                    {% set final_p = calculate_electricity_price(item.price, as_datetime(item.start)) %}
                    {% set ns.items = ns.items + [final_p | float] %}
                    {% endfor %}
                {% endif %}
                {{ ns.items }}

    - sensor:
        - name: "Current Hourly Electricity Price with Tariffs"
        unique_id: "sensor.current_hourly_electricity_price_with_tariffs"
        unit_of_measurement: "EUR/kWh"
        state: >
            {% from 'electricity.jinja' import calculate_electricity_price %}
            {# Get the current raw hourly price from Nordpool #}
            {% set raw = states('sensor.nord_pool_lt_dabartine_kaina') | float(0) %}
            {# If the hourly sensor is in EUR/kWh already, multiply by 1000 to feed the macro #}
            {{ calculate_electricity_price(raw * 1000, now()) }}

    - sensor:
        - name: "Current 15min Electricity Price with Tariffs"
        unique_id: "sensor.current_15min_electricity_price_with_tariffs"
        unit_of_measurement: "EUR/kWh"
        state: >
            {% set prices = state_attr('sensor.electricity_prices_today', 'prices') %}
            {% if prices and prices | length >= 96 %}
                {# Calculate current 15-min index (0-95) #}
                {% set idx = (now().hour * 4) + (now().minute // 15) %}
                {{ prices[idx] | float(0) | round(3) }}
            {% else %}
                {# Fallback to the hourly sensor only if the 15-min list isn't ready #}
                {{ states('sensor.current_hourly_electricity_price_with_tariffs') }}
            {% endif %}                 

```

Based on an outdated Nordpool blueprint.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FBrunas%2FHomeAssistant%2Fmaster%2Fblueprints%2Fautomation%2Fawtrix_electricity_prices.yaml)