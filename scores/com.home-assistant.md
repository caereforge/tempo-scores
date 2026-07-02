# Home Assistant

Home Assistant is a home automation platform. Tempo shows state changes for the entities you choose.

## Helper

**Not required.** Home Assistant posts to Tempo directly with a `rest_command`.

## Setup

Home Assistant emits an event for almost everything, so curate at the source: send Tempo only the entities you mean to watch.

**1. Tag the entities you want.** In Home Assistant, add a **label** (for example `tempo`) to each entity you want on the timeline.

**2. Define the `rest_command`** in `configuration.yaml`. The score matches on the keys `entity_id`, `device_class`, `friendly_name`, `area` and `state`, so the payload must send those exact metadata keys:

```yaml
# configuration.yaml
rest_command:
  tempo_ingest:
    url: "http://<your-mac-ip>:7776/ingest"
    method: POST
    content_type: "application/json"
    headers:
      X-Tempo-Token: "<token>"
    payload: >
      {"title": "{{ friendly_name }} -> {{ state }}",
       "providerIdentifier": "com.home-assistant",
       "metadata": {
         "entity_id": "{{ entity_id }}",
         "device_class": "{{ device_class }}",
         "friendly_name": "{{ friendly_name }}",
         "area": "{{ area }}",
         "state": "{{ state }}",
         "new_state": "{{ new_state }}",
         "old_state": "{{ old_state }}"
       }}
```

Copy `<token>` from **Settings > Ingestion**. Reload the YAML configuration (Developer Tools > YAML > Reload, or restart Home Assistant) so the new command is registered.

**3. Add an automation** that fires only for the labeled entities and passes those keys to the command. `device_class` comes from the entity's attributes via `state_attr(...)`:

```yaml
trigger:
  - platform: event
    event_type: state_changed
condition:
  - "{{ trigger.event.data.entity_id in label_entities('tempo') }}"
action:
  - service: rest_command.tempo_ingest
    data:
      entity_id: "{{ trigger.event.data.entity_id }}"
      device_class: "{{ state_attr(trigger.event.data.entity_id, 'device_class') }}"
      friendly_name: "{{ trigger.event.data.new_state.attributes.friendly_name }}"
      area: "{{ area_name(trigger.event.data.entity_id) }}"
      state: "{{ trigger.event.data.new_state.state }}"
      new_state: "{{ trigger.event.data.new_state.state }}"
      old_state: "{{ trigger.event.data.old_state.state }}"
```

A real working payload carries `device_class`, `domain`, `entity_id`, `friendly_name`, `new_state`, `old_state` and `state`. Without `device_class` and `entity_id` an event lands at the `info` default: the smoke / CO / leak / alarm / lock styling depends on those keys.

## What you'll see

A state change per labeled entity, with its friendly name, new state and area. The score raises severity for safety device classes (`smoke`, `carbon_monoxide`, `gas`, `moisture`, `safety` critical; `tamper`, `problem`, `battery` warning) and for `alarm_control_panel.*` / `lock.*` entity states.

The score's default actions: Open dashboard, Open entity history, Open automations, Copy entity ID, HA docs. The dashboard URLs default to `http://homeassistant.local:8123` (edit them in the Score Editor for a different host).

## Known limitations

Curation lives on the Home Assistant side: the `tempo` label decides what is sent. If the timeline fills up, tighten the label rather than the score.
