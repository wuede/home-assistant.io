---
title: Nord Pool
description: Instructions on how to integrate with the Nord Pool Energy market prices.
ha_category:
  - Energy
  - Finance
  - Sensor
ha_release: 2024.12
ha_iot_class: Cloud Polling
ha_config_flow: true
ha_codeowners:
  - '@gjohansson-ST'
ha_domain: nordpool
ha_platforms:
  - sensor
ha_integration_type: integration
---

The **Nord Pool** {% term integration %} integrates [Nord Pool Group](https://www.nordpoolgroup.com/) energy prices into Home Assistant.

The {% term integration %} provides the public market prices displayed on the [Nord Pool Auction page](https://data.nordpoolgroup.com/auction/day-ahead/prices).

{% include integrations/config_flow.md %}

{% tip %}
Only a single configuration entry is supported. Use the reconfigure option from the configuration entry if needed to modify the settings.

EUR is the base currency for market prices. If you choose another currency, you can find the conversion rate in the `Exchange rate` sensor.
All prices are displayed as `selected_currency/kWh`.
{% endtip %}

{% configuration_basic %}
Areas:
  description: Select one or multiple market areas to create sensors for.
Currency:
  description: Currency to display prices in. EUR is the base currency in Nord Pool prices.
{% endconfiguration_basic %}

## Sensors

Each market area will create a device which has the following sensors:

### Main sensors

- Current price for the selected area
- Previous price for the selected area
- Next price for the selected area

These sensors can be used to calculate your current energy cost, or to help decide whether to charge the battery now or in an hour, etc.

### Block price sensors

- Block average
- Block minimum
- Block maximum
- Block start time
- Block end time

These sensors show the minimum/maximum and average during certain blocks of the day. More known as off-peak (typically lower price) or peak hours (typically higher price).
The block price sensors are not enabled by default.

### Daily average

- Daily average

The daily average sensor is not enabled by default.

### Diagnostic sensors

- Last updated - indicates when the market price was last updated.
- Currency - The selected currency.
- Exchange rate - EUR is the base currency so will show the exchange rate used on the market place.

The `Exchange rate` sensor is not enabled by default.

## Custom actions

### Get price for date

As the integration entities only provides price information for the current date, use the custom action "Get price for date" to retrieve pricing information for the last two months up until tomorrow.

Providing the areas and currency are optional, and will use the integration configured if omitted.

See [examples](#examples) how to use in a trigger template sensor.

{% note %}

The public api only allows to see past pricing information up to 2 months.

Tomorrow's prices are typically released around 13:00 CET, and therefore trying to get tomorrow's prices before that time will generate an error which needs to be considered in such use-case.

{% endnote %}

#### Example action with data

{% raw %}

```yaml
action: nordpool.get_prices_for_date
data:
  config_entry: 1234567890a
  date: "2024-11-10"
  areas:
    - SE3
    - SE4
  currency: SEK
```

{% endraw %}

## Examples

A simple template sensor to add VAT and a fixed cost from an `input_number` entity

{% raw %}

```yaml
template:
  - sensor:
      - name: "Full SE3 current price"
        unit_of_measurement: "SEK/kWh"
        state_class: measurement
        state: >
          {% set cost = states('sensor.nord pool_se3_current_price') | float(0) %}
          {% set add_cost = states('input_number.add_fixed_cost') | float(0) %}
          # Add fixed cost to the spot price and add VAT (25%)
          {{ ((cost + add_cost) * 1.25) | round(2, default=0) }}
```

{% endraw %}

Use a trigger template entity to see tomorrow's lowest price.

{% raw %}

```yaml
template:
  - trigger:
      - trigger: time_pattern
        minutes: /1
    action:
      - action: nordpool.get_prices_for_date
        data:
          config_entry: 1234567890A
          date: "{{ now().date() + timedelta(days=1) }}"
          areas:
            - SE3
          currency: SEK
        response_variable: tomorrow_price
    sensor:
      - name: Tomorrow lowest price
        unique_id: se3_tomorrow_low_price
        state: >
          {% set data = namespace(prices=[]) %}
          {% for state in tomorrow_price['SE3'] %}
            {% set data.prices = data.prices + [state.price] %}
          {% endfor %}
          {{min(data.prices)}}
        attributes:
          data: "{{ tomorrow_price['SE3'] }}"
```

{% endraw %}
