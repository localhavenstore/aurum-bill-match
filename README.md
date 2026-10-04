# Aurum Bill Match (free)

Home Assistant's Energy dashboard only knows a **price per kWh**. Your real bill also has a **standing charge**
(a fixed amount per day), so the dashboard's cost never matches the bill. Bill Match adds the standing charge.

Built only with Home Assistant's own features (helpers and one trigger-based template sensor):
**no custom integration, no HACS plugin, no fake second meter, nothing loaded from the internet.**

> Money numbers are **estimates** made from the tariff you type in. Bill Match cannot read your contract; check the
> result against your next bill.

## What you get
- **`sensor.bill_match_total_cost`** - energy cost + standing charge, counted from the day you start. Pick it in the
  Energy dashboard as the grid cost entity (below) and the dashboard shows the cost **including** the standing charge.
  Attributes: `energy_part`, `standing_part`, `counting_since`, `fee_log` (the last 10 standing charge changes) and
  `problems` (what is not set up yet, in plain words).
- **`sensor.bill_match_energy_cost`** - the energy part alone, with `price_log` (the last 10 price changes) and
  `jump_log` (meter jumps that were not counted).
- 5 helpers you fill in once: standing charge per day, price per kWh, the grid meter, an optional price entity, a start
  date - and a **"new meter"** button for the day your meter is replaced.

## Install (about 5 minutes)
1. Make sure `configuration.yaml` loads packages (add it once, then check the configuration):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy `packages/bill_match.yaml` to `/config/packages/` (create the folder if needed).
   Not paying in EUR? Open the file and change the word `EUR` in the line marked `CURRENCY` to your Home Assistant
   currency (for example `GBP`).
3. Restart Home Assistant.
4. Settings > Devices & services > Helpers, search "Bill Match" and fill in:
   - **standing charge per day** and **price per kWh** - from your bill (use the amounts you actually pay, so
     including VAT if your bill adds VAT);
   - **grid meter (entity id)** - your grid import energy sensor, for example `sensor.grid_import_energy`
     (kWh, Wh or MWh, the same sensor the Energy dashboard uses);
   - **price entity** - optional: an entity with a changing price, in your currency or its cents per kWh, MWh or Wh
     (EUR: `EUR/kWh`, `EUR/MWh`, `ct/kWh`; GBP: `GBP/kWh`, `p/kWh`; USD: `USD/kWh`, `c/kWh` ...). Other units - for
     example pence while your currency is EUR - are refused and shown in `problems`. Leave it empty to use the fixed price;
   - **start date** - counting begins on this date, or today if the date is in the past.
5. Settings > Dashboards > Energy > your grid consumption > **Use an entity tracking the total costs** >
   `sensor.bill_match_total_cost`. Done.

Check `sensor.bill_match_total_cost` > attributes > `problems`: an empty list means everything is set up.

Requirements: Home Assistant 2026.9 or newer (tested on 2026.9.4; older versions not tested), a grid import energy sensor.

## How it counts
- **Energy:** once a minute (and whenever you change a helper) Bill Match looks at the meter and the price. Each
  increase of the meter is multiplied by the price Bill Match saw at its check **just before** the increase showed up.
  With a changing price this is exact as long as Bill Match's once-a-minute check sees a new meter value between two
  price changes - true for meters that report every few seconds or minutes and prices that change every 15 minutes or
  more. Everything the meter added between two checks is priced at one price (the latest seen before); so is the step
  of a meter that reports only every 30 minutes, or that was unavailable.
- **Standing charge:** every day from the start to today (today included) is charged once. If Home Assistant was off
  at midnight, the missed days are added at the next start - each at the standing charge that was in effect.
- **Changes:** a new price counts from that moment on. A higher standing charge counts from today (today is topped up
  to the new amount - never charged twice), a lower one from tomorrow. Past days keep their value, so the total **never jumps down** and the Energy
  dashboard never shows a negative cost.
- **Meter resets and jumps:** if the meter drops below 90 % of the last reading (reset, new meter), Bill Match starts
  again from the new value - nothing negative, no jump. Smaller dips are ignored. A rise of more than **250 kW on
  average** since the meter last moved (for example a new meter that starts at a higher number) is not counted either: it
  is written to `jump_log` and shown in `problems`. Replacing your meter? Press **"Bill Match - new meter"** after the
  new meter reports; counting starts again from its reading.
- **Unavailable meter:** nothing is counted; the next good reading catches up (priced as described above).
- **No back-fill:** Bill Match cannot see your meter's past, so it starts counting on the start date (or today).
- Want it to count right now (for example in a test)? Fire the event `bill_match_refresh`.

## Tested
On Home Assistant 2026.9.4, in a separate throwaway test instance with a synthetic meter (never a real home):
a 30-day month with changing prices (total = sum of kWh x price + 30 x standing charge, to the cent), a restart in
the middle (nothing lost, nothing counted twice), meter reset / small dip / unavailable / new meter, standing charge
changes, a price entity in EUR/MWh, and the Energy dashboard accepting the total as the cost entity (no validation
issues, long-term statistics written). Test log: `TEST_RESULT.txt`.

## FAQ
**Why does the cost go up a little just after midnight?** That is the new day's standing charge: it is added once, in
the first minute of the day (on the first day: when you set Bill Match up).
**I have day/night prices.** Use a price entity that changes with the time of day (for example a template sensor), or the
Pro edition's levies + VAT on top of it.
**My bill also has a monthly fee, levies or VAT.** The Pro edition adds those (below). In the free edition, enter
amounts that already include VAT.
**How do I undo it?** Delete `packages/bill_match.yaml` and restart; in the Energy dashboard switch the cost back to
"Use a static price" or "Do not track costs". Your meter and its history are never changed.
**How do I update?** Replace `packages/bill_match.yaml` with the new version and restart - the totals continue.
**Can I rename the entities?** Yes - Bill Match finds its two sensors by an internal role attribute, so the totals
continue. Keep only one copy of the package.
**Does it fill my database?** While your meter value changes: up to 2 small state rows per minute (like any energy
sensor that updates every minute). With an unchanged meter there are no minute-by-minute writes - only the daily
standing charge (once a day) and your own helper changes update the sensors. Home Assistant's normal history clean-up
removes old rows; the long-term statistics stay small.

## Pro edition
**[Aurum Bill Match Pro](https://antrikos.gumroad.com/l/aurum-bill-match)** (Gumroad, EUR 7) replaces this file and keeps your history. It adds:
- **monthly and yearly fixed fees** (spread over the days) and **monthly/yearly credits**,
- up to **3 per-kWh levies** and **VAT** (on energy, and on fixed fees if you want),
- your **bill period** (monthly, every 2 or 3 months, yearly - on your bill's start day) with **"this bill so far"**
  and a **forecast** to the end of the period,
- a **"my last bill was ..." check**: the difference to what Bill Match counted, and which number to change by how much,
- example presets (UK / NL / DE / GR: VAT rate, VAT on fees, bill period - prices always from your own bill) and a
  ready dashboard card.

**Known limit (Pro 1.0.0):** if your meter reports rarely (for example every 15-30 minutes), the energy of one
report interval around midnight can land in the next bill period. The totals stay exact; only that interval moves
between two bills (usually a few cents). A fix is planned for 1.0.1.

## Licence
MIT - see [LICENSE](LICENSE). Not affiliated with Home Assistant or the Open Home Foundation.
Made with AI assistance and tested by us.
