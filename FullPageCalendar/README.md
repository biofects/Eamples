[![Sponsor Me](https://img.shields.io/badge/Sponsor%20Me-%F0%9F%92%AA-purple?style=for-the-badge)](https://github.com/sponsors/biofects?frequency=recurring&sponsor=biofects)


# 🌐 Full Page Calendar with Add Event card.

## 🔍 About

This Home Assistant tutorial on creating a full page calendar using the native Local Calendar integration of home Assistant. I put this together as I did not want sidebar showing and could not modify the look the of the default. So I have put together some automations and yaml to give me the look that fits my dashboard. Please note I may turn this into a plugin in the near future.


---
## 💸 Donations Appreciated!
If you find this tutorial useful, please consider donating. Your support is greatly appreciated!

### Sponsor me on GitHub
[![Sponsor Me](https://img.shields.io/badge/Sponsor%20Me-%F0%9F%92%AA-purple?style=for-the-badge)](https://github.com/sponsors/biofects?frequency=recurring&sponsor=biofects) 

### or
## Paypal

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=TWRQVYJWC77E6)
---


---

## ✨ Requirements, Theme and Integratrions I use to make this possible. 
#### Requirements 
- [Local calendar](https://www.home-assistant.io/integrations/local_calendar/)
#### Theme
- [Enhanced-Biofects](https://github.com/biofects/Enhanced-Biofects)
#### Integrations
- [Sidebar Card](https://github.com/DBuit/sidebar-card)
- [WallPanel](https://community.home-assistant.io/t/wallpanel-addon-wall-panel-mode-for-your-home-assistant-dashboards/449857)
- [lovelace-card-mod](https://github.com/thomasloven/lovelace-card-mod)


## 🏗️ Configurations and Scripts
You will need to understant how to add scripts and input conrtrols in your home assistnat. I will try to be very clear on how to add each one. My configuration might not be exacly as yours, I will use the defaults that is provided when installing Home Assitant.

#### Input Datetime
You will need to edit your `configuration.yaml` and add the following.

```yaml
input_datetime:
  biofects_all_day_date:
    name: Event Date
    has_date: true
    has_time: false
    initial: "2025-01-01"

  biofects_event_start:
    name: Start Time
    has_date: true
    has_time: true
    initial: "2025-01-01 08:00:00"

  biofects_event_end:
    name: End Time
    has_date: true
    has_time: true
    initial: "2025-01-01 09:00:00"

input_text:
  biofects_event_title:
    name: Event Title
    initial: ""

  biofects_event_description:
    name: Event Description
    initial: ""

  biofects_event_notification:
    name: Event Creation Notification
    initial: ""

input_boolean:
  biofects_all_day:
    name: All Day Event
    initial: false

input_select:
  biofects_event_repeat:
    name: Repeat
    options:
      - Never
      - Daily
      - Weekly
      - Monthly Date
      - Monthly nth Weekday
      - Yearly
    initial: Never

  biofects_event_nth_week:
    name: Repeat nth Week
    options:
      - 1st
      - 2nd
      - 3rd
      - 4th
      - Last
    initial: 1st

  biofects_event_weekday:
    name: Repeat Weekday
    options:
      - Monday
      - Tuesday
      - Wednesday
      - Thursday
      - Friday
      - Saturday
      - Sunday
    initial: Monday


```

#### Create Event automations
Inside your scripts.yaml you need to add this in order to publish the new event to your calendar.
```yaml
create_event:
  alias: Create Event
  description: Create calendar events with validation and auto-reset date/time
  mode: single
  variables:
    calendar_entity: calendar.my_calendar
    all_day: "{{ is_state('input_boolean.biofects_all_day', 'on') }}"
    event_title: "{{ states('input_text.biofects_event_title') | trim }}"
    event_description: "{{ states('input_text.biofects_event_description') }}"
    repeat_type: "{{ states('input_select.biofects_event_repeat') }}"
    repeat_count: "{{ states('input_number.biofects_event_repeat_count') | int }}"
    monthly_day: "{{ states('input_number.biofects_event_repeat_day') | int }}"
    monthly_nth_week: "{{ states('input_select.biofects_event_nth_week') }}"
    monthly_weekday: "{{ states('input_select.biofects_event_weekday') }}"
    start_date: >-
      {{ states('input_datetime.biofects_event_start') if not all_day else
      states('input_datetime.biofects_all_day_date') }}
    end_date: >-
      {{ states('input_datetime.biofects_event_end') if not all_day else
      states('input_datetime.biofects_all_day_date') }}
  sequence:
    - choose:
        - conditions:
            - condition: template
              value_template: "{{ event_title == '' }}"
          sequence:
            - service: input_text.set_value
              target:
                entity_id: input_text.biofects_event_notification
              data:
                value: "Error: Event title is required."
            - stop: Missing required fields.
        - conditions:
            - condition: template
              value_template: "{{ not all_day and (start_date == '' or end_date == '') }}"
          sequence:
            - service: input_text.set_value
              target:
                entity_id: input_text.biofects_event_notification
              data:
                value: "Error: Start and end time are required."
            - stop: Missing required fields.
        - conditions:
            - condition: template
              value_template: >-
                {{ not all_day and (start_date | as_datetime) >= (end_date |
                as_datetime) }}
          sequence:
            - service: input_text.set_value
              target:
                entity_id: input_text.biofects_event_notification
              data:
                value: "Error: End time must be after start time."
            - stop: Invalid time range.
        - conditions:
            - condition: template
              value_template: "{{ repeat_type != 'Never' and repeat_count < 1 }}"
          sequence:
            - service: input_text.set_value
              target:
                entity_id: input_text.biofects_event_notification
              data:
                value: "Error: Repeat count must be at least 1."
            - stop: Invalid repeat count.
    - repeat:
        count: "{{ repeat_count }}"
        sequence:
          - variables:
              next_event_date: >
                {% set base_start = start_date | as_datetime %} {% if repeat_type
                == 'Daily' %}
                  {{ base_start + timedelta(days=repeat.index - 1) }}
                {% elif repeat_type == 'Weekly' %}
                  {{ base_start + timedelta(weeks=repeat.index - 1) }}
                {% elif repeat_type == 'Monthly Date' %}
                  {% set new_month = base_start.month + (repeat.index - 1) %}
                  {% set new_year = base_start.year + ((new_month - 1) // 12) %}
                  {% set adjusted_month = (new_month - 1) % 12 + 1 %}
                  {{ base_start.replace(year=new_year, month=adjusted_month, day=monthly_day) }}
                {% elif repeat_type == 'Monthly nth Weekday' %}
                  {% set weekdays = {'Monday': 0, 'Tuesday': 1, 'Wednesday': 2, 'Thursday': 3, 'Friday': 4, 'Saturday': 5, 'Sunday': 6} %}
                  {% set week_map = {'1st': 0, '2nd': 1, '3rd': 2, '4th': 3, 'Last': -1} %}
                  {% set new_month = base_start.month + (repeat.index - 1) %}
                  {% set new_year = base_start.year + ((new_month - 1) // 12) %}
                  {% set adjusted_month = (new_month - 1) % 12 + 1 %}
                  {% set first_day = base_start.replace(year=new_year, month=adjusted_month, day=1) %}
                  {% set target_weekday = weekdays[monthly_weekday] %}
                  {% set target_week = week_map[monthly_nth_week] %}
                  {{ first_day.replace(day=1 + (7 * target_week) + ((target_weekday - first_day.weekday() + 7) % 7)) }}
                {% elif repeat_type == 'Yearly' %}
                  {{ base_start.replace(year=base_start.year + (repeat.index - 1)) }}
                {% else %}
                  {{ base_start }}
                {% endif %}
          - service: calendar.create_event
            target:
              entity_id: "{{ calendar_entity }}"
            data:
              summary: "{{ event_title }}"
              description: "{{ event_description }}"
              start_date_time: >
                {% set start_dt = next_event_date | as_datetime %} {% if all_day
                %}
                  {{ start_dt.strftime('%Y-%m-%dT00:00:00') }}
                {% else %}
                  {{ start_dt.strftime('%Y-%m-%dT%H:%M:%S') }}
                {% endif %}
              end_date_time: >
                {% set end_dt = (next_event_date | as_datetime) + (end_date |
                as_datetime - start_date | as_datetime) %} {% if all_day %}
                  {{ end_dt.strftime('%Y-%m-%dT23:59:59') }}
                {% else %}
                  {{ end_dt.strftime('%Y-%m-%dT%H:%M:%S') }}
                {% endif %}
    - service: input_text.set_value
      target:
        entity_id: input_text.biofects_event_notification
      data:
        value: Event Created Successfully!
    - delay: "00:00:03"
    - service: input_text.set_value
      target:
        entity_id: input_text.biofects_event_notification
      data:
        value: ""
    - service: input_text.set_value
      target:
        entity_id: input_text.biofects_event_title
      data:
        value: ""
    - service: input_text.set_value
      target:
        entity_id: input_text.biofects_event_description
      data:
        value: ""
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.biofects_all_day
    - service: input_select.select_option
      target:
        entity_id: input_select.biofects_event_repeat
      data:
        option: Never
    - service: input_number.set_value
      target:
        entity_id: input_number.biofects_event_repeat_count
      data:
        value: 1
    - service: input_number.set_value
      target:
        entity_id: input_number.biofects_event_repeat_day
      data:
        value: 1
    - service: input_select.select_option
      target:
        entity_id: input_select.biofects_event_nth_week
      data:
        option: 1st
    - service: input_select.select_option
      target:
        entity_id: input_select.biofects_event_weekday
      data:
        option: Monday
```

#### This is to get current datetime to display on your card to add event
In your scripts.yaml file add this to populate the current date Time on your form, else it will default to 2025-01-01 00:00:00
```yaml
initialize_event_datetime_fields:
  alias: Initialize Event Datetime Fields
  mode: single
  sequence:
    - service: input_datetime.set_datetime
      target:
        entity_id: input_datetime.biofects_all_day_date
      data:
        date: "{{ now().strftime('%Y-%m-%d') }}"
    - service: input_datetime.set_datetime
      target:
        entity_id: input_datetime.biofects_event_start
      data:
        datetime: "{{ now().replace(second=0, microsecond=0).isoformat() }}"
    - service: input_datetime.set_datetime
      target:
        entity_id: input_datetime.biofects_event_end
      data:
        datetime: "{{ (now() + timedelta(hours=1)).replace(second=0, microsecond=0).isoformat() }}"
```
#### Lovelace Dashboard
Please note this is my lovelace dashboard including sidebar configurations to fit my theme using card mode integrations.
```yaml
sidebar:
  style: |
    :host {
      --sidebar-background-color: rgba(30, 34, 44, 0.18);
      --sidebar-text-color: var(--primary-text-color, #b2ebf2);
      --sidebar-icon-color: var(--primary-text-color, #b2ebf2);

      background-color: var(--sidebar-background-color) !important;
      color: var(--sidebar-text-color);
      backdrop-filter: blur(18px) saturate(140%) contrast(1.08);
      -webkit-backdrop-filter: blur(18px) saturate(140%) contrast(1.08);
      border: 2px solid rgba(0, 188, 212, 0.32);
      border-right: 2px solid var(--accent-color, #00bcd4); /* strong right border */
      box-shadow: 2px 0 8px 0 rgba(0,0,0,0.10); /* subtle shadow for separation */
      margin-right: 8px; /* ensures border is visible */
      padding: 8px 0 8px 0;
      font-family: var(--primary-font-family, 'Roboto', 'Segoe UI', Arial, sans-serif);
      border-radius: 18px 0 0 18px;
      transition: background 0.3s, box-shadow 0.3s;
    }
    .sidebar-inner {
      position: relative;
      height: 100%;
    }
    .digitalClock, .date {
      text-align: center;
      color: var(--sidebar-text-color);
      font-size: 13px;
      text-shadow: 0 1px 8px rgba(0, 188, 212, 0.18);
      letter-spacing: 0.5px;
    }
    .sidebarMenu li {
      display: flex;
      align-items: center;
      justify-content: flex-start;
      line-height: 26px;
      padding: 7px 14px 7px 10px;
      border-radius: 10px;
      margin: 2px 6px;
      border-bottom: none;
      color: var(--sidebar-text-color);
      transition: background 0.2s, color 0.2s;
    }
    .sidebarMenu li .icon {
      color: var(--sidebar-icon-color);
    }
    .sidebarMenu li.active {
      background-color: var(--sidebar-selected-background-color);
      color: var(--sidebar-selected-text-color);
      box-shadow: 0 2px 8px 0 rgba(0, 188, 212, 0.08);
    }
    .sidebarMenu li:hover {
      /* No hover effect for tablets */
    }
    .bottom {
      margin-top: 48% !important;
      width: 75% !important;
      padding: 0 10px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    .bottom ha-card {
      width: 100% !important;
      max-width: 100% !important;
      background-color: rgba(10, 20, 40, 0.7) !important;
      border: 1px solid rgba(0, 188, 212, 0.10) !important;
      border-radius: 10px !important;
      box-shadow: 0 2px 10px rgba(0, 188, 212, 0.08) !important;
      margin-bottom: 10px;
      overflow: hidden !important;
    }
    .bottom .card-content {
      width: 70% !important;
      padding: 10px !important;
    }
    /* Make the first card in bottom (select box) shorter */
    .bottom ha-card:first-child {
      height: auto !important;
      min-height: 0 !important;
    }
    .bottom ha-card:first-child .card-content {
      padding: 7px !important;
    }
    /* Media player specific styling */
    .bottom .type-media-control ha-card {
      background-color: rgba(10, 20, 40, 0.7) !important;
    }
    .bottom .media-player {
      --mdc-theme-primary: var(--sidebar-text-color) !important;
      --paper-slider-active-color: var(--sidebar-text-color) !important;
      --paper-slider-knob-color: var(--sidebar-text-color) !important;
      --paper-slider-pin-color: var(--sidebar-text-color) !important;
    }
    .bottom .media-player mwc-icon-button {
      color: var(--sidebar-text-color) !important;
    }
    .bottom .entity-select, 
    .bottom select {
      color: var(--sidebar-text-color) !important;
      background-color: rgba(10, 30, 50, 0.7) !important;
      border-color: rgba(0, 188, 212, 0.15) !important;
    }
  width:
    mobile: 0
    tablet: 12
    desktop: 12
  digitalClock: true
  date: true
  dateFormat: DD-MMM-YYYY
  twelveHourVersion: true
  period: true
  clock: false
  hideTopMenu: false
  hideHassSidebar: false
  sidebarMenu:
    - action: navigate
      navigation_path: /main-menu
      name: Menu
      icon: mdi:menu
      active: true
    - action: navigate
      navigation_path: /dashboard-home
      name: Home
      icon: mdi:home
      active: true
    - action: navigate
      navigation_path: /dashboard-climate
      name: Climate
      icon: mdi:thermometer-lines
      active: true
    - action: navigate
      navigation_path: /dashboard-irrigation
      name: Irrigation
      icon: mdi:sprinkler-variant
      active: true
    - action: navigate
      navigation_path: /dashboard-lights
      name: Lights
      icon: mdi:lightbulb
      active: true
    - action: navigate
      navigation_path: /dashboard-office
      name: Office
      icon: mdi:chair-rolling
      active: true
    - action: navigate
      navigation_path: /dashboard-security
      name: Security
      icon: mdi:cctv
      active: true
    - action: navigate
      navigation_path: /dashboard-calendar
      name: Calendar
      icon: mdi:calendar
      active: true
  bottomCard:
    type: grid
    cardOptions:
      columns: 1
      square: false
      cards:
        - type: button
          show_name: true
          show_icon: true
          icon: mdi:microphone
          name: Talk to Optimus
###### Removed a new feature not ready, its a mic button to start conversation with AI
views:
  - title: Home
    type: sections
    max_columns: 4
    sections:
      - type: custom:stack-in-card
        cards:
          - type: custom:layout-card
            layout_type: custom:grid-layout
            layout:
              grid-template-columns: 70% 30%
              grid-template-rows: auto
              grid-template-areas: |
                "greeting weather"
            cards:
              - type: markdown
                content: >
                  {% set hour = now().hour %} {% set greeting = 
                    'Good Morning' if 4 <= hour < 12 else
                    'Good Afternoon' if 12 <= hour < 17 else
                    'Good Evening'
                  %} # {{ greeting }} Crazy Family <br> <div style="font-size:
                  2.0em; color: var(--secondary-text-color);">
                    {{ state_attr('sensor.daily_quote_feed', 'entries')[0].summary }} - {{ state_attr('sensor.daily_quote_feed', 'entries')[0].title }}
                  </div>
                card_mod:
                  style: |
                    ha-card {
                      box-shadow: none;
                      border: none;
                      padding: 8px !important;
                    }
                grid_area: greeting
              - type: weather-forecast
                entity: weather.forecast_home
                show_current: false
                show_forecast: true
                forecast_type: daily
                card_mod:
                  style: |
                    ha-card {
                      box-shadow: none;
                      border: none;
                      padding: 8px !important;
                      margin: 0px !important;
                      text-align: right;
                    }
                grid_area: weather
        card_mod:
          style: |
            ha-card {
              background-color: var(--card-background-color);
              border-radius: 12px;
              padding: 0px !important;
            }
        column_span: 4
      - type: grid
        cards:
          - type: calendar
            entities:
              - calendar.my_calendar
              - calendar.my404_gmail_com
            grid_options:
              columns: full
              rows: auto
            card_mod:
              style: |
                ha-full-calendar {
                  --calendar-height: 70vh !important;
                }
        column_span: 3
      - type: grid
        cards:
          - type: entities
            title: Create Event
            entities:
              - type: conditional
                conditions:
                  - entity: input_text.biofects_event_notification
                    state_not: ''
                row:
                  entity: input_text.biofects_event_notification
                  name: Notification
              - entity: input_text.biofects_event_title
                name: Event Title
              - entity: input_boolean.biofects_all_day
                name: All Day Event
              - type: conditional
                conditions:
                  - entity: input_boolean.biofects_all_day
                    state: 'on'
                row:
                  entity: input_datetime.biofects_all_day_date
                  name: Event Date
              - type: conditional
                conditions:
                  - entity: input_boolean.biofects_all_day
                    state: 'off'
                row:
                  entity: input_datetime.biofects_event_start
                  name: Start Date & Time
              - type: conditional
                conditions:
                  - entity: input_boolean.biofects_all_day
                    state: 'off'
                row:
                  entity: input_datetime.biofects_event_end
                  name: End Date & Time
              - entity: input_text.biofects_event_description
                name: Description
              - entity: input_select.biofects_event_repeat
                name: Repeat
              - type: conditional
                conditions:
                  - entity: input_select.biofects_event_repeat
                    state_not: Never
                row:
                  entity: input_number.biofects_event_repeat_count
                  name: Repeat Count
              - type: conditional
                conditions:
                  - entity: input_select.biofects_event_repeat
                    state: Monthly Date
                row:
                  entity: input_number.biofects_event_repeat_day
                  name: Day of Month
              - type: conditional
                conditions:
                  - entity: input_select.biofects_event_repeat
                    state: Monthly nth Weekday
                row:
                  entity: input_select.biofects_event_nth_week
                  name: Nth Week
              - type: conditional
                conditions:
                  - entity: input_select.biofects_event_repeat
                    state: Monthly nth Weekday
                row:
                  entity: input_select.biofects_event_weekday
                  name: Weekday
              - type: custom:button-card
                name: Add Event
                tap_action:
                  action: call-service
                  service: script.create_event
                styles:
                  card:
                    - background-color: var(--primary-button-background-color)
                    - border-radius: var(--ha-card-border-radius)
                    - box-shadow: 0px 0px 10px var(--primary-glow-color)
                    - padding: 10px
                    - height: 50px
                    - display: flex
                    - justify-content: center
                    - align-items: center
                  name:
                    - font-size: var(--primary-font-size)
                    - font-weight: bold
                    - color: var(--primary-text-color)
                  icon:
                    - color: var(--primary-text-color)
                    - width: 24px
                    - height: 24px
              - type: custom:button-card
                name: 🔄 Reset to Now
                tap_action:
                  action: call-service
                  service: script.initialize_event_datetime_fields
                styles:
                  card:
                    - background-color: var(--secondary-background-color)
                    - border-radius: var(--ha-card-border-radius)
                    - padding: 8px
                    - height: 40px
                    - display: flex
                    - justify-content: center
                    - align-items: center
                    - margin-bottom: 10px
                  name:
                    - font-size: 12px
                    - color: var(--secondary-text-color)
```


## 📜 License

All my integrations and tutorials are licensed under the MIT License - see a LICENSE file for details.

## ⚖️ Disclaimer

I am not responsible for any damage, data loss, or adverse issues that may occur as a result of following this tutorial or implementing any of the instructions provided. Please proceed at your own risk and ensure you have proper backups before making changes to your Home Assistant setup.
