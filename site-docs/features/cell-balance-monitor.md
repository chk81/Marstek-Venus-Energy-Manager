# Cell balance monitor

Tracks the voltage spread between the strongest and weakest battery cell after each weekly full charge, giving you a long-term picture of how well your battery cells are staying in balance.

## How to enable

The balance monitor is enabled in the **Weekly full charge** configuration step (initial wizard or options flow). Enabling it also bypasses the solar charge delay on the weekly full charge day so the battery can reach the top, run active balancing, and then take the formal OCV reading.

## How it works

### Normal high-SOC charge protection

This protection is always active during automatic charging. It is not an active
recovery mode and it does not force the battery to charge. It only limits or
pauses charging when a battery is already near the top of its SOC/voltage range,
so normal daily operation does not make cell imbalance worse.

The logic is evaluated per battery. In a multi-battery system, one battery can be
paused or tapered while another battery continues charging normally.

#### Charge power limits

| Condition for one battery | Maximum charge allocation for that battery |
|---|---:|
| `max_cell_voltage` below 3.45 V | Normal configured limit |
| `max_cell_voltage` at or above 3.45 V | 100 W |
| `max_cell_voltage` at or above 3.55 V | Charging paused for measurement |

SOC is intentionally ignored in this path. The protection is based only on cell
voltage so an incorrect SOC report cannot start or stop the top-charge logic.

#### Voltage taper latch

When `max_cell_voltage` reaches 3.45 V, the battery enters the 100 W voltage
taper. This taper is latched while the battery remains in the high-voltage zone,
so charge power does not jump back up just because the cell voltage briefly
settles below 3.45 V.

The latch is cleared only after `max_cell_voltage` falls below 3.45 V.

#### Pause and final discharge

If `max_cell_voltage` reaches 3.55 V, charging is paused for that battery. The
controller waits 60 seconds, records `delta_V = Vmax - Vmin`, then discharges the
battery at 25 W until `max_cell_voltage` falls to 3.42 V. After that final
discharge the battery is released back to normal control.

#### Daily high-voltage exposure limit

For each battery, the controller tracks how long it has spent in the top-voltage
zone during the current day. The zone is counted when `max_cell_voltage` is at
or above 3.45 V. After 4 hours in that zone on the same day, normal automatic
charging is no longer extended for that battery.

### Active top-balancing profile

Active top-balancing is used in these situations:

- **Normal max SOC set to 100 %**: per battery, when that battery reaches the
  3.55 V measurement point.
- **Weekly full charge day or manual full-charge trigger**: globally, after all
  participating batteries reach the top by BMS cutoff or `max_cell_voltage >= 3.55 V`.
- **Active balance mode**: per battery, when the per-battery switch is enabled.

Once active, the controller uses voltages in V throughout:

| Condition for one battery | Command |
|---|---:|
| Before `max_cell_voltage >= 3.45 V` | Normal PD charging to the top |
| `max_cell_voltage >= 3.45 V` | Enter regulated balance charge |
| Regulated charge until `max_cell_voltage >= 3.55 V` | Charge at 50 W |
| At `max_cell_voltage >= 3.55 V` | Stop and wait 60 s |
| After wait, `delta_V > 0.03 V` | Discharge at 25 W to 3.49 V, then retry charge |
| After wait, `delta_V <= 0.03 V` | Final discharge at 25 W to 3.42 V and finish |
| If the BMS rejects charge | Continue discharging and lower the retry voltage by 0.01 V |

The active-control logic measures `delta_V = max_cell_voltage - min_cell_voltage`.
It does not convert this value to mV. The active mode no longer stops after 48
hours; it runs until `delta_V <= 0.03 V` after the 3.55 V measurement point, or
until the user disables the switch. The current phase is persisted so the mode
continues in the correct phase after a Home Assistant restart.

#### Diagnostics

The **Integration Status** sensor exposes a `normal_balance_protection`
attribute with per-battery details:

| Attribute | Meaning |
|---|---|
| `in_zone` | Whether the battery is currently in the top-balancing zone |
| `exposure_h` | Hours spent in that zone today |
| `daily_limit_h` | Current daily exposure limit, normally 4 h |
| `paused` | Whether charging is currently paused by high cell voltage |
| `max_cell_voltage` / `min_cell_voltage` | Current cell voltage extremes |
| `delta_V` | Current voltage spread between highest and lowest cell |
| `voltage_taper_latched` | Whether the 100 W voltage taper is latched |
| `active_balance_phase` | Current active-balancing phase, if any |
| `charge_limit_w` | Effective per-battery charge limit before allocation |
### OCV reading sequence (weekly full charge day)

After the weekly full charge has completed its active top-balancing phase, the
cell balance monitor can still take the formal OCV reading used for long-term
health tracking. At that point the integration:

1. **Holds discharge** â€” prevents the battery from discharging so the cells can rest under no-load conditions.
2. **Waits 15 minutes** â€” allows BMS active balancing to settle and surface voltages to stabilise.
3. **Checks stability** â€” requires at least 5 consecutive polls with power below 50 W and voltage change below 5 mV between polls.
4. **Takes the reading** â€” records `delta_mV = (Vmax âˆ’ Vmin) Ã— 1000`.
5. **Releases discharge** â€” unless the result is orange (see thresholds below).

### Orange hold (2.5-hour passive balancing)

If the reading lands in the orange zone (100â€“149 mV), discharge remains blocked for 2.5 hours to let passive balancing work. After the hold period a follow-up reading is taken and discharge is released regardless of the result.

### Opportunistic readings

On days other than the weekly full charge day, if the battery is already at 100 % SOC and power is already below 50 W, the integration takes a lightweight reading without blocking discharge. Limited to once every 24 hours.

## Thresholds

| Status | Delta range | Meaning |
|---|---|---|
| ðŸŸ¢ Green | < 50 mV | Good balance |
| ðŸŸ¡ Yellow | 50 â€“ 99 mV | Minor imbalance â€” monitor over time |
| ðŸŸ  Orange | 100 â€“ 149 mV | Moderate imbalance â€” 2.5 h passive balancing hold initiated |
| ðŸ”´ Red | â‰¥ 150 mV | High imbalance |

Thresholds are fixed and apply equally to all LFP cell chemistries.

## Notifications

The integration sends Home Assistant persistent notifications for the following events:

| Event | Notification title |
|---|---|
| Orange or red reading | âš ï¸ Cell imbalance â€” {battery name} |
| Orange persists after 2.5 h hold | âš ï¸ Cell imbalance persists â€” {battery name} |
| Red on 2 or more consecutive charges | ðŸ”´ Possible degraded cell â€” {battery name} |
| Rising trend with average above 75 mV | ðŸ“ˆ Rising imbalance trend â€” {battery name} |

## Sensor entities

Five sensor entities are created per battery when the feature is enabled:

| Entity | Description | Unit |
|---|---|---|
| `sensor.*_cell_delta` | Voltage spread between max and min cell | mV |
| `sensor.*_balance_status` | Balance result: `green` / `yellow` / `orange` / `red` | â€” |
| `sensor.*_delta_trend` | Trend over the last formal readings: `rising` / `stable` / `falling` | â€” |
| `sensor.*_last_balance_read` | Timestamp of the last reading | timestamp |
| `sensor.*_delta_avg_4w` | Rolling average of the last 4 formal readings | mV |

Values are restored from persistent storage after a Home Assistant restart so sensors show the last known state immediately on startup.

## Technical notes

- The voltage spike visible at 100 % SOC (before the wait period) is normal BMS active balancing behaviour â€” not a real imbalance. The 15-minute wait ensures the reading is taken at true open-circuit voltage.
- Up to 52 readings are stored per battery (approximately one year of weekly charges).
- The 4-week average and trend are calculated from formal readings only (not opportunistic), so they reflect the pattern at true open-circuit voltage.

!!! info
    Cell voltage registers (`max_cell_voltage`, `min_cell_voltage`) are read from all supported battery versions (v2, v3, vA, vD).
