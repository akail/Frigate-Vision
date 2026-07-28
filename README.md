# Frigate-Vision
---
<sup>**Firstly, credit where credit is due!** A lot of this automation was inspired and even copied from both @SgtBatten's [Frigate notification Blueprint](https://community.home-assistant.io/t/frigate-mobile-app-notifications-2-0/559732) and @valentinfrlch's [LLMVision Blueprint](https://llmvision.gitbook.io/getting-started/setup/blueprint). If anyone has an issue with content or code, please reach out.</sup>

[After sharing a screenshot](https://www.reddit.com/r/homeassistant/comments/1lohkx9/my_take_on_a_frigatellm_vision_notification/) of one of my Frigate automations the other day, a few of you asked if I had a blueprint. At the time I didn’t… so I sat down, taught myself how to build one and here it is!

Introducing **Frigate Vision**; a blueprint designed to bring intelligent notifications and AI object recognition to your Home Assistant setup, powered by Frigate, LLMVision, and Home Assistant.

> **Note:** This fork sends notifications over **Signal only**. Home Assistant mobile app support has been removed; if you want push notifications to a phone, use [the original](https://community.home-assistant.io/t/frigate-mobile-app-notifications-2-0/559732) instead.

---

**📄 Get the Blueprint:**

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fakail%2FFrigate-Vision%2Fblob%2Fmain%2Ffrigate_vision.yaml)

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/zacharyd3)
#### 💡 What Frigate Vision Does

* **🚨 Listens for new Frigate detection events** from any camera you choose using MQTT
* **🧠 Integrates with LLMVision** to enrich notifications with event summaries
* **🕒 Enforces per-camera cooldowns** so you’re not spammed when a squirrel does laps in your yard
* **💬 Sends Signal messages** with custom text, camera names, and optional sublabels (e.g., who or what was recognized)
* **🖼️ Attaches the event thumbnail** directly to the message, with a link to the full clip in the body
* **🧩 Uses input helpers** so you can easily reuse this blueprint across cameras without editing YAML

---

### 🛠️ Why I Built It:
I’ve used @SgtBatten’s reworked Frigate notifications for years and finally decided to break it down and recreate it to my liking. It started out as a fun project, but once I discovered **LLMVision** and the ability to generate dynamic event summaries from clips, I was *hooked*.

I’ve since spent time crafting what I felt was the ultimate smart notification setup for me. This fork drops the mobile app path entirely and delivers everything over Signal. This is still a **beta version**, but it’s what I’d call *mostly complete*.


---

### ⚙️ Requirements:

* [Frigate installed](https://docs.frigate.video/integrations/home-assistant/) with MQTT events enabled
* [LLMVision](https://llmvision.org/) installed and configured
* [Signal Messenger](https://www.home-assistant.io/integrations/signal_messenger/) configured as a notify platform
* An input_boolean helper for multi-camera queuing
* A publicly reachable Home Assistant URL, so the clip links in messages resolve

---

### 💬 Signal Notifications

Signal is the only notification channel. You get up to three messages per event: initial detection, mid-event updates, and the final AI summary.

**Nothing is sent unless you fill in *Signal Service Name*.** Listing recipients on their own does nothing.

#### Setup

The core Signal Messenger integration is configured in YAML and registers a `notify.<name>` action:

```yaml
# configuration.yaml
notify:
  - name: signal
    platform: signal_messenger
    url: "http://127.0.0.1:8080"
    number: "+15555550100"          # your Signal number
    recipients:
      - "+15555550199"              # or a Group ID
```

In the blueprint, set **Signal Service Name** to `signal` (the `name:` you used above, with or without the `notify.` prefix). That's it, no recipients needed; messages go to the `recipients:` already configured on the platform. Use **Signal Notification Recipients** only when you want to override that list for this automation.

Set **Home Assistant Public URL** to a domain reachable from outside your LAN. It builds the `Clip:` link in every message.

#### Why is this a text field and not a dropdown?

Signal Messenger is a *legacy* notify platform: it registers a `notify.<name>` action but creates no entity. Home Assistant has no selector that can enumerate notify actions ([WTH is there no notify selector](https://community.home-assistant.io/t/wth-is-there-no-notify-selector/467457)), and an entity picker filtered to the `notify` domain would list unrelated things like your phone while never showing Signal. So the service name is typed by hand.

#### Images and clips

The **thumbnail is a genuine attachment** — Home Assistant downloads it and hands the bytes to Signal, so it renders inline. The **clip is a link**, not an attachment: `signal_messenger` caps downloads at 50 MB and raises past that, which would take the whole message down with it. A link always works regardless of clip length.

---

### ⏱️ What arrives, and when

```
detection  ->  [1] initial message      thumbnail + clip link
               [2] update messages      sent whenever the snapshot or
                                        sublabel changes, 0 or more times
  event end
     +30s  ->  LLMVision analyses the clip (queued behind other cameras
               via the input_boolean helper, 3 minute max wait)
           ->  [3] AI summary           thumbnail + clip link
           ->  cooldown delay
```

Two behaviours worth knowing:

* The blueprint runs `mode: single`, and the cooldown is a delay at the *end* of the run. The automation is therefore busy for the whole event plus analysis plus cooldown, and new events on that camera are dropped during that window. Use one automation per camera.
* If another camera holds the LLMVision lock for more than 3 minutes, the run stops before the AI summary. You still get messages [1] and [2], but not [3].

### 🧠 TL;DR:

**Frigate Vision** makes your HA notifications smarter by blending real-time detection with AI object recognition and smart logic; all wrapped in one reusable, configurable blueprint.

---

I’d love your feedback, ideas, bug reports (via github please), and feature requests. If you use it and like it, drop a comment; let’s make Frigate Vision even smarter!
