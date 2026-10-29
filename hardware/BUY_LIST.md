# Buy list

Prices are approximate 2026 US retail (Amazon, Adafruit, Digi-Key, Mouser, Pimoroni; whichever is
convenient). Everything here is low-voltage and commodity; exact brands don't matter unless noted.
"Qty" includes spares, because transistors from the same batch differ in avalanche voltage and you
will want to try a few.

## A. Required (skip anything you already own)

| # | Item | Qty | Approx. cost | Notes |
|---|---|---|---|---|
| 1 | **Raspberry Pi Pico H** (RP2040, headers pre-soldered) | 2 | $6 each | One to use, one spare / second independent unit. Pico 2 is fine too. Avoid "Pico W" (Wi-Fi radio is a noise source you don't need). |
| 2 | **Micro-USB cable, data-capable** | 1 | $5 | Many cheap cables are charge-only. Short (≤1 m) is better for noise. |
| 3 | **Solderless breadboard**, 830 tie points | 1 | $7 | Half-size (400) also works but gets cramped with the tin. |
| 4 | **Jumper wire kit** (male-male, assorted lengths) | 1 kit | $6 | Pre-cut solid-core wires lie flat and pick up less interference than long flexible jumpers. |
| 5 | **2N3904 NPN transistors** | 10 | $3 for 10 | The noise source *and* the gain stage. BC547 or 2N2222 also work; 2N3904 is the one documented in the build guide. |
| 6 | **Resistor kit**, ¼ W, assorted 10 Ω–1 MΩ | 1 kit | $8 | Needed values: 100 kΩ (bias), 10 kΩ, 4.7 kΩ, 2.2 kΩ, 1 kΩ, 470 Ω, 100 Ω, 47 Ω. |
| 7 | **Capacitor kit**, ceramic (1 nF–1 µF) + electrolytic (10–470 µF) | 1 kit | $9 | Needed: 100 nF ×6, 10 nF ×2, 1 nF ×2, 10 µF ×2, 100 µF ×1. |
| 8 | **9 V batteries** | 2 | $8 for 2 | Two in series = 18 V, dropped through 100 kΩ to bias the junction. Batteries, not a wall adapter, so there is no mains ripple to argue about. |
| 9 | **9 V battery snap connectors** (with leads) | 2 | $4 | |
| 10 | **Metal tin** (Altoids-style) | 1 | $3 or free | The shield. A small steel electrical box also works. |
| 11 | **Digital multimeter** | 1 | $15–25 | Any basic one. Used to measure the actual bias voltage and check the gain stage's DC operating point. |

**Subtotal A: about $70–80** if you own none of it; **about $25** if you already have a breadboard, wires, kits and a meter.

## B. Strongly recommended

| # | Item | Qty | Approx. cost | Notes |
|---|---|---|---|---|
| 12 | **Oscilloscope** (see options below) | 1 | $0–70 | Lets you *see* the noise, set the gain so it fills the ADC range, and spot mains hum before you spend 35 minutes capturing it. |
| 13 | **MCP6002 or MCP6022 op-amp** (DIP-8, rail-to-rail, single-supply) | 2 | $1–2 each | Optional alternative gain/buffer stage. The default build uses a second 2N3904, which is faster; the op-amp version is easier to bias predictably. Buy both, decide on the bench. |
| 14 | **Alkaline AA ×2 + holder** or a **12 V "A23" battery + holder** | 1 | $5 | Alternative bias supply giving ~12 V instead of 18 V. Not needed if you buy the 9 V pair. |

### Oscilloscope options
| Option | Cost | Good enough? |
|---|---|---|
| None; use the Pico itself plus the quick-look script as a slow scope | $0 | Workable. The diagnostics script shows histogram, spectrum and a time-series snippet; you lose the ability to see MHz-scale noise or debug the circuit live. |
| Handheld pocket scope (FNIRSI DSO-TC3 / 1C15 class, ~0.5–1 MHz bandwidth) | $35–45 | Adequate for checking DC levels, hum, and "is there noise at all". Too slow to see the true noise bandwidth. |
| USB scope, 20 MHz, 2-channel (Hantek 6022BE class) | $60–75 | Recommended if you want one. Sees the real noise bandwidth, lets you compare the junction output with the ADC input. |

## C. Optional / later

| Item | Approx. cost | Why |
|---|---|---|
| USB isolator (ADuM3160/4160-based) | $12–25 | Breaks the ground loop to the PC. Only worth it if Control B (source off) shows PC-borne noise. |
| Small perfboard/protoboard + 22 AWG solid wire + soldering iron | $15–40 | For a permanent version once the breadboard circuit works. Not needed for the experiment itself. |
| 1 % metal-film 100 kΩ resistor | $1 | Marginally more stable bias current than a 5 % carbon film. Cosmetic. |
| Second Pico set up as a logic-analyser/scope (`sigrok` PicoScope-style firmware) | $6 | Cheap way to get a slow scope if you buy none. |

## Totals

| Scenario | Approx. total |
|---|---|
| Own nothing, no scope | $70–80 |
| Own nothing, pocket scope | $110–125 |
| Own nothing, USB scope | $140–155 |
| Own the usual hobby kit (meter, breadboard, wires, kits) | $25–35 (+ scope if wanted) |

## Safety (short version; the build guide has the full one)
Everything runs below 20 V DC from batteries. The only things that can bite: electrolytic capacitors
inserted backwards (they vent), shorting a 9 V battery through a jumper (gets hot), and plugging
the 18 V bias into a Pico pin (kills the Pico, which is why there is a spare). Keep the bias
battery on its own corner of the breadboard and unplug it when not capturing.
