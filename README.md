# Amazon.it Tracking Codes
This project will retrieve Amazon tracking codes from notification email and put them into Home Assistant ```todo``` entity.

## Let's start
Firts of all you need to setup the IMAP integration in Home Asisstant. If you are using Google with 2FA, you need to setup an app password as specified [here](https://www.home-assistant.io/integrations/imap/).

- Put your imap details, like user/password/server and port.
- In the search box put:  ```FROM conferma-spedizione@amazon.it UnSeen UnDeleted```
- Make sure to check Body ```text``` in ```event_message_data```
- Once created, click on CONFIGURE button and type ```{{text}}``` into ```custom template```.

## Automation
Let's assume you have a todo entity called ```todo.amazon_trackcodes```.

You can create an automation like this:

```
alias: add_amazon_tracking_codes
description: ""
triggers:
  - trigger: event
    event_type: imap_content
    id: custom_event
conditions: []
actions:
  - target:
      entity_id: todo.amazon_trackcodes
    data:
      status: needs_action
    response_variable: codes
    action: todo.get_items
  - variables:
      order_number: >
        {{ (trigger.event.data["custom"] | regex_findall("Ordine:
        #\s*(\d{3}-\d{7}-\d{7})(?:.|\n)*spedizione è: (\w+)"))[0][0] }}
      new_code: >
        {{ (trigger.event.data["custom"] | regex_findall("Ordine:
        #\s*(\d{3}-\d{7}-\d{7})(?:.|\n)*spedizione è: (\w+)"))[0][1] }}
  - if:
      - condition: template
        value_template: >-
          {{ new_code not in codes['todo.amazon_trackcodes']['items'] |
          map(attribute='summary') | list }}
    then:
      - action: todo.add_item
        metadata: {}
        data:
          item: |
            {{new_code}}
          description: |
            {{order_number}}|{{trigger.event.data["subject"]}}
        target:
          entity_id: todo.amazon_trackcodes
      - action: notify.domus
        metadata: {}
        data:
          message: "{{order_number}}|{{trigger.event.data['subject']}}"
mode: single
```

Of course, you can add ```condition``` or ```event_data``` to this automation to better filter imap events.

This is just an example. You can just be notified about the new shipping without adding to the _todo list_ or you can adapt to your needs.

## Throubleshooting and test

To test if all is ok, you can look for an email into your inbox caming from _conferma-spedizione@amazon.it_ and mark as unread, then you can reload the integration to force imap event to fire again:

```
action: homeassistant.reload_config_entry
data: {}
target:
  entity_id: sensor.imap_<name_of_the_sensor_created>
```

