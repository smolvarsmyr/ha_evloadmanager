# EV Load Manager for Home Assistant

Smart EV charging control for Home Assistant, built for Norwegian households on
**Norgespris**. It's written for a **Zaptec** installation and a **Tibber Pulse**
meter, but it will work with any meter that reports phase currents and power.

It protects your main fuse, keeps each clock hour under a kWh limit (to keep
your *kapasitetsledd* down), and only charges in the cheap night/weekend period
unless you press **Charge now**.

It's a plain Home Assistant **package**: one YAML file of helpers, template
sensors and automations, plus one dashboard YAML. You don't need any custom
integration for EVLM itself.

---

## What it does

| Constraint | How it works |
|---|---|
| **Main fuse protection** | Watches the actual current on L1/L2/L3 at the main supply. If the highest phase goes above *main fuse − safety margin*, the Zaptec installation's *Available current* drops right away by roughly the overshoot. |
| **Hourly energy limit** | Tracks kWh used in the current clock hour, projects where the hour will end at the current power draw, and reduces charging if the projection goes over your limit. |
| **Price control** | Each charger only charges in the cheap period (by default weekdays 22–06 and all weekend). Outside that, the charger's *max current* is set to 0 A. |
| **Charge now** | A button per charger that ignores price control for 12 hours. Fuse and hourly protection stay on. |

### Design principles

- **Feedback, not modelling.** EVLM doesn't try to work out which Zaptec phase
  maps to which house phase. It changes the installation current, watches the
  meter, and corrects again.
- **Fast down, slow up.** Reductions happen right away. Increases happen one
  *recovery step* at a time, with a cooldown between steps, and only when both
  the fuse and the hourly budget have spare margin. The exception is the start
  of a new clock hour: the budget resets, so EVLM jumps straight up to whatever
  the fuse headroom allows.
- **Fail safe.** If a protection is enabled and its measurements become
  unavailable, the installation current is set to **0 A**.
- **Two separate levers.** The installation *Available current* (shared by all
  chargers) is only used for fuse and hourly protection. Each charger's *max
  current* is only used for price scheduling. They never fight each other.
- **Swappable price layer.** All price logic sits in a few template sensors
  (`binary_sensor.evlm_price_allowed` is the key one). You can swap them for a
  spot-price planner later without touching the load controller.

### How it decides (every minute and on every meter update)

```
measurements missing?            -> set 0 A (fail-safe)
fuse or hourly target < current  -> reduce immediately
new clock hour, budget reset     -> jump up to fuse headroom
fuse + hour have margin, cooldown passed
                                 -> increase by one recovery step
otherwise                        -> hold
```

Every action is written to the logbook and to `input_text.evlm_last_action`
along with the reason.

---

## Requirements

- Home Assistant 2024.10 or newer (uses the `triggers:` / `actions:` syntax)
- **Zaptec** integration, which provides the installation *Available current*
  and per-charger *Charger max current* number entities
- A meter that provides:
  - current per phase (A) for L1, L2, L3
  - whole-house power (W)
  - cumulative energy (kWh, ever-increasing)

  The **Tibber** integration with a Pulse provides all of these.
- [Packages](https://www.home-assistant.io/docs/configuration/packages/) enabled
  in `configuration.yaml`

The package is set up for **two chargers** called *Garage* and *Outside*.

---

## Installation

### 1. Enable packages

In `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 2. Add the package

Copy `ev_load_manager.yaml` to `config/packages/ev_load_manager.yaml`.

### 3. Point it at your entities

Open the file and edit the **CONFIGURATION** section at the top:

```yaml
attributes:
  meter_current_l1: &meter_current_l1 sensor.tibber_pulse_your_home_current_l1
  ...
  installation_current: &installation_current number.your_installation_available_current
  garage_max_current: &garage_max_current number.your_garage_charger_charger_max_current
  ...
```

Only change the entity ID after the `&anchor`. Each entity ID appears once. The
rest of the file picks it up through YAML anchors (`*meter_current_l1`) or by
reading the `sensor.evlm_config` attributes, so you shouldn't need to edit
anything below that section.

| Key | What to put there |
|---|---|
| `meter_current_l1/l2/l3` | Main-supply current per phase, in A |
| `meter_power` | Whole-house power, in **W** |
| `meter_energy` | Cumulative energy, in kWh (feeds an hourly `utility_meter`) |
| `installation_current` | Zaptec installation *Available current* (number) |
| `garage_max_current` / `outside_max_current` | Each charger's *Charger max current* (number) |
| `price_cheap` / `price_expensive` | Total price per kWh. Only used for display |
| `cheap_from_hour` / `cheap_until_hour` | Cheap period, e.g. 22 → 6. Must cross midnight |
| `cheap_weekends` | `true` if the whole weekend is cheap |

### 4. Restart and check

Restart Home Assistant. Then check `sensor.evlm_configuration_check`: it should
say **OK**. If it doesn't, it lists the configured entities that are missing or
have no value.

### 5. Set the helpers before enabling control

Input numbers start at their minimum the first time, and every switch starts
**off**. Set these first (from the dashboard or *Settings → Helpers*):

| Helper | Suggested starting point |
|---|---|
| Main fuse | Your main fuse rating, e.g. 63 A |
| Fuse safety margin | 3 A |
| Fuse recovery buffer | 2 A |
| Maximum energy per clock hour | Just under your next *kapasitetsledd* step, e.g. 9.5 kWh |
| Hourly recovery buffer | 0.2 kWh |
| Maximum installation current | Your Zaptec installation's rating, e.g. 32 A |
| Minimum useful charging current | 6 A |
| Estimated kW reduction per amp | 0.23 kW/A (conservative) |
| Minutes between current increases | 5 min |
| Normal recovery step | 3 A |
| Garage / Outside charger maximum | 32 A (or your charger's limit) |

Then turn on **main fuse protection**, **hourly energy limit** and **price
control**, and finally **automatic control**. With automatic control off, EVLM
only monitors. All the sensors and the dashboard still work, so you can watch
what it would do before letting it act.

### 6. Add the dashboard (optional)

1. Create a new dashboard, open it, click the pencil, then **⋮ → Raw configuration editor**.
2. Paste in `dashboard.yaml`.
3. Edit the `evlm_entities:` block at the top in the same way (installation
   current plus the Zaptec sensors for each charger).

The dashboard uses the *sections* layout. It shows live status, the main fuse
and hourly-energy state with graphs, per-charger scheduling and Charge now
buttons, the price schedule, and all tunable settings.

> Home Assistant resolves the anchors when you save, so you won't see them if
> you reopen the raw editor. Keep this file as your source and paste it again
> after changes.

---

## Entities created

All entities are prefixed `evlm_`. The main ones:

| Entity | Purpose |
|---|---|
| `input_boolean.evlm_automatic_control` | Master switch. Off means monitoring only |
| `input_boolean.evlm_fuse_guard` / `_hourly_cap` / `_price_control` | Turn each constraint on or off |
| `sensor.evlm_status` | What the controller is doing right now |
| `input_text.evlm_last_action` | Last change and why |
| `sensor.evlm_configuration_check` | Shows whether all configured entities exist |
| `sensor.evlm_highest_phase_current` / `_limiting_phase` / `_fuse_headroom` | Fuse state |
| `sensor.evlm_energy_used_this_hour` / `_projected_energy_this_hour` / `_allowed_house_power_this_hour` | Hourly budget state |
| `sensor.evlm_fuse_target_current` / `_hourly_target_current` / `_reduction_target` | What each constraint wants the installation current to be |
| `binary_sensor.evlm_price_allowed` | On during the cheap period |
| `input_button.evlm_*_charge_now` / `timer.evlm_*_charge_now` | 12-hour Charge now override per charger |
| `sensor.evlm_phase_l1_current` … `sensor.evlm_installation_current` | Internal mirrors of your configured source entities |

---

## Customising

- **One charger:** remove the *outside* entries (input_number, input_button,
  timer, binary/price-status sensors, the OUTSIDE block in the price-control
  automation, the Charge now automation branch, and the dashboard cards), or
  point both at the same charger.
- **Other charger names:** *Garage* and *Outside* are only labels. Change the
  `name:` fields to whatever suits you. Leave the entity IDs alone unless you
  rename them everywhere.
- **Different meter:** any meter works as long as it provides per-phase current
  in A, power in W and cumulative energy in kWh. If your power sensor reports
  kW, remove the `/ 1000` in `sensor.evlm_house_power`.
- **Spot pricing:** replace the state template of `binary_sensor.evlm_price_allowed`
  (for example with a "cheapest N hours" sensor). Everything else follows from it.
- **Override duration:** Charge now lasts 12 hours. Change `duration` in the
  `timer:` section and in the Charge now automation.

---

## About Norgespris and kapasitetsledd

*Norgespris* is the Norwegian fixed electricity price offered through the grid
companies. Your total price still varies with your grid company's day/night
tariff, and that difference is what the price schedule follows. Put your own
numbers in `price_cheap` / `price_expensive` and the matching hours.

The *kapasitetsledd* (capacity charge) in the Norwegian grid tariff is based on
your highest hourly consumption peaks each month. Capping the kWh per clock hour
keeps EV charging from pushing you into a more expensive step.

---

## Disclaimer

This software controls high-power equipment. It is **not** a replacement for
correctly rated fuses, breakers or the charger's own load balancing. Check the
behaviour on your own installation, start with automatic control off, and use
it at your own risk.
