# Add a BIQU MicroProbe to a QIDI Q1 Pro and keep the nozzle probe

The Q1 Pro ships with two Z sensors. Most MicroProbe guides replace both. This guide
replaces only the one that fails, and keeps the one that makes the printer
self-calibrating.

Tested on a QIDI Q1 Pro running QIDI's Klipper fork, version `25ee65122`
(firmware V4.4.21), on 2026-09-25.

## What you get

| | Standard MicroProbe guides | This guide |
|---|---|---|
| Bed mesh sensor | MicroProbe | MicroProbe |
| Z=0 sensor | MicroProbe, fixed `z_offset` | Nozzle piezo, measured every print |
| Calibrate `z_offset` after a nozzle change | Yes | No |
| Calibrate `z_offset` after changing the probe mount | Yes | No |
| Detects a blob on the nozzle | No | Yes, as a number in the log |

The difference matters because a fixed `z_offset` is a guess that ages. Change the
nozzle, knock the probe, or leave a bead of plastic on the tip, and the first layer
moves with no warning. The piezo measures the real nozzle-to-plate distance before
every print, so those changes correct themselves.

## Who this is for

You need to be comfortable editing Klipper config files over SSH or through the
Fluidd or Mainsail file editor. You do not need to understand Klipper internals.
Every file and line is named.

Back up these files before you start. Copy them somewhere off the printer.

```
~/klipper_config/printer.cfg
~/klipper_config/gcode_macro.cfg
~/klipper_config/Adaptive_Mesh.cfg
```

## How the Q1 Pro measures Z

The printer has two separate sensors, on two separate pins:

| Sensor | Config section | Pin | Job |
|---|---|---|---|
| Piezo, under the bed | `[smart_effector]` | `U_1:PC1` | Finds Z=0 by touching the plate with the nozzle |
| Inductive, on the toolhead | `[qdprobe]` | `!gpio21` | Probes the bed mesh |

QIDI's fork adds two commands, `QIDI_PROBE_PIN_1` and `QIDI_PROBE_PIN_2`, that switch
which sensor Klipper reads. `PIN_1` selects `[smart_effector]`. `PIN_2` selects
`[qdprobe]`.

`[qdprobe]` depends on a probe object that `[smart_effector]` creates. The two sections
work as a pair. Neither works alone.

Every print runs this sequence:

1. `G28` homes Z roughly, against whichever sensor is selected.
2. `get_zoffset` selects the piezo, touches the plate with the nozzle five times, and
   declares that contact point to be Z = -0.07.
3. `G29` selects the inductive sensor and probes the bed mesh.

Step 2 is what makes the printer self-calibrating. Step 3 is the one that fails, because
an inductive sensor measures the metal under the plate rather than the surface the
plastic lands on.

## Why the usual approach fails

Most guides delete both sections and add a single `[probe]` section for the MicroProbe.
That removes the piezo, so `get_zoffset` no longer exists and Z=0 comes from a fixed
`z_offset` that you calibrate by hand.

On the test machine, hand calibration with the paper method produced `1.530`. The piezo
later measured the same distance as `1.743`. The error of `0.213 mm` put Z=0 almost
double the intended distance above the plate, which ruined every first layer until the
piezo came back.

The single-probe approach has two more problems on this fork:

- `SAVE_CONFIG` does not persist `[probe] z_offset`. Three calibrations wrote `0.000`
  every time. You have to set the value by hand and delete the auto-generated block.
- `G29` normalizes the mesh with `save_meshoffset` and `set_meshoffset`, which assume
  `z_offset` is near zero. Give `[probe]` a real value and the mesh gets corrected twice,
  which drives the nozzle into the plate.

Keeping the piezo avoids all of this. There is nothing to calibrate and nothing to
persist.

## Step 1: wire the probe

Connect the MicroProbe signal wire to the **existing inductive probe connector** on the
toolhead. That connector is `gpio21`. Leave the piezo wiring alone; it runs to the main
board on a different pin and this conversion never touches it.

Connect the MicroProbe control wire to `gpio11`. On the Q1 Pro that header is labeled
for a second hotend fan and is not populated, so the pin is free. Confirm this on your
own machine: heat the nozzle above 50°C and check that both heatsink fans still spin.

## Step 2: edit printer.cfg

### Replace the probe sections

Find `[qdprobe]` in `~/klipper_config/printer.cfg`. Replace the `[smart_effector]` and
`[qdprobe]` sections with these. Keep everything else in the file as it is.

```ini
[smart_effector]
pin: U_1:PC1
recovery_time: 0
x_offset: 24
y_offset: 10
z_offset: 0.000001
speed: 5
lift_speed: 5
probe_accel: 50
samples: 2
samples_result: submaxmin
sample_retract_dist: 5.0
samples_tolerance: 0.05
samples_tolerance_retries: 5

[qdprobe]
pin: ^!gpio21
z_offset: 0.000001
activate_gcode:
    probe_deploy
    G4 P500
deactivate_gcode:
    probe_stow
deactivate_on_each_sample: false
```

Three changes from the QIDI originals:

- **`x_offset` and `y_offset` change from `17.6` and `4.4` to `24` and `10`.** These
  values live on `[smart_effector]`, but Klipper applies them to whichever sensor is
  active. Only the MicroProbe probes the mesh, so they must be the MicroProbe's offsets.
  Measure your own mount rather than copying these.
- **`[qdprobe] pin` gains a `^`.** The MicroProbe needs the internal pull-up resistor
  that the inductive sensor did not.
- **`[qdprobe]` gains `activate_gcode` and `deactivate_gcode`.** These deploy and retract
  the pin around each probing run. The original QIDI sensors had no moving parts and
  needed neither.

Leave `z_offset: 0.000001` on both sections. That value is deliberate. The piezo sets the
real zero at print time, so Klipper must not apply an offset of its own.

### Add the control pin

Add this section anywhere in `printer.cfg`:

```ini
[output_pin probe_enable]
pin: gpio11
value: 1
```

`value: 1` means the pin starts in the retracted position. The pin must be retracted
when the piezo probes, or the pin touches the plate before the nozzle does.

### Check the Z endstop

Find `[stepper_z]` and confirm this line is present and not commented out:

```ini
position_endstop: -0.23
```

Klipper refuses to start without it. Some MicroProbe guides tell you to comment it out,
which is correct for a single `[probe]` section and wrong here.

## Step 3: add the deploy macros

Add these to `~/klipper_config/gcode_macro.cfg`:

```ini
[gcode_macro probe_deploy]
gcode:
    SET_PIN PIN=probe_enable VALUE=0

[gcode_macro probe_stow]
gcode:
    SET_PIN PIN=probe_enable VALUE=1
```

Check the direction before you print. Run `probe_deploy` and watch the pin extend, then
`probe_stow` and watch it retract. If the pin moves the wrong way, swap the two `VALUE`
numbers.

## Step 4: leave the QIDI macros alone

Do not modify `PRINT_START`, `homing_override`, `G29`, `get_zoffset`, `set_zoffset`,
`save_zoffset`, `move_subzoffset`, or `test_zoffset`. They work unchanged. They call
`QIDI_PROBE_PIN_1` and `QIDI_PROBE_PIN_2`, which still exist because `[qdprobe]` is still
there.

If a guide told you to delete or comment out any of those macros, restore them from your
backup.

One optional change removes a duplicate homing cycle. `PRINT_START` calls `G28`, and
`G29` then calls `G28` again. To home once per print and still allow `G29` on its own,
wrap the call inside `G29`:

```ini
{% if 'xyz' not in printer.toolhead.homed_axes %}
    G28
{% endif %}
```

## Step 5: adjust the nozzle wipe

`CLEAR_NOZZLE` in `gcode_macro.cfg` wipes the nozzle by moving it left and right across
a brush. The stock travel reaches `X65`, which collides with the MicroProbe.

Change every `G1 X65` and `G1 X85` in `CLEAR_NOZZLE` to `G1 X77`. Verify the limit on
your own machine by moving the toolhead there by hand before you run the macro.

## Step 6: set the mount height

The MicroProbe pin has a fixed travel. On the test machine that travel is `2.36 mm`, and
it divides between two things that both matter:

```
deployed reach + stowed clearance = pin travel
```

- **Deployed reach** is how far the pin hangs below the nozzle when extended. It must
  exceed anything stuck to the nozzle tip, or the nozzle reaches the plate first and the
  probe never fires.
- **Stowed clearance** is how far the pin sits above the nozzle when retracted. It must
  exceed the height of anything already printed, or the pin catches on the part.

Aim for at least `1.0 mm` deployed and at least `0.6 mm` stowed. The test machine runs
`1.60` and `0.76`.

Raising the probe in its mount converts deployed reach into stowed clearance, one
millimeter for one millimeter. If your mount clamps the probe body, set the height there
rather than by redesigning the bracket. Clamp position moves the result more than
bracket geometry does.

## Step 7: verify

Run each step and check the result before moving on.

1. **Restart Klipper.** It must report ready with no warnings.

2. **Confirm the switching commands exist.** Send `QIDI_PROBE_PIN_2`, then
   `QIDI_PROBE_PIN_1`. Both return silently. `Unknown command` means `[qdprobe]` did not
   load.

3. **Home the printer.** Send `G28` and watch the pin. It deploys, probes, and retracts.
   Only the MicroProbe runs here.

4. **Measure the nozzle zero.** Heat the bed to 70°C and the nozzle to 150°C, then send
   `get_zoffset`. Stand at the machine. The nozzle descends and touches the plate five
   times. Read the `Result is z=` line in the console.

   That number is the MicroProbe's standoff, measured by the nozzle. Record it. On the
   test machine it is `-1.601`. Five samples should agree within about `0.01 mm`.

5. **Check Z=0.** Send `G1 X120 Y120 F6000` then `G1 Z0 F300`. The nozzle sits `0.07 mm`
   above the plate, because `get_zoffset` declares the contact point as `-0.07`. A
   `0.05 mm` feeler gauge drags. Paper, at about `0.1 mm`, does not fit.

6. **Check the stowed clearance.** Send `probe_stow`, return to `Z0`, and slide feeler
   gauges under the **pin**. Subtract `0.07` from whatever fits.

7. **Print something small** and look at the first layer.

After this, the `Result is z=` value appears in `klippy.log` on every print. Watch it. A
drop of more than about `0.1 mm` means something is on the nozzle tip.

## Optional: guard against a bad mesh

An inductive or mechanical probe can misfire and produce a mesh that drives the nozzle
into the plate or lifts it clear of it. On the test machine, one misfire produced a mesh
that lifted the nozzle `0.954 mm` and wasted seven meters of filament before anyone
noticed.

A guard macro reads the mesh after `G29`, compares it against limits, and stops the print
if the numbers are implausible. A healthy mesh on this machine has a range of `0.03` to
`0.17 mm`. The broken one had a range of `1.175 mm`, so the two are easy to tell apart.

See `mesh_guard.cfg` in this repository. Add `[include mesh_guard.cfg]` to `printer.cfg`
above the `#*# <---------------------- SAVE_CONFIG ---------------------->` line, then
add `VALIDATE_MESH` to `PRINT_START` immediately after `G29`.

Anything below that `SAVE_CONFIG` line gets rewritten automatically, and this printer
rewrites it on every print. Config you put there disappears.

## Optional: fix the purge line

KAMP probes only the area your part occupies, and it draws the purge line outside that
area. The purge therefore prints on ground the printer never measured, which makes it the
least reliable line of the print.

In `~/klipper_config/Adaptive_Mesh.cfg`:

| Line | Setting | Change to |
|---|---|---|
| 24 | `variable_margin_enable` | `True` |
| 25 | `variable_margin_size` | `15` |
| 168 | `variable_distance_to_object_y` | `5` |

The mesh then extends 15 mm past the part, and the purge line moves to 5 mm from it. The
purge lands on measured ground with 10 mm to spare.

Both of the first two settings are required. Line 62 of the same file forces
`margin_size` to `0` whenever `margin_enable` is `False`.

## Troubleshooting

**`Unknown command: QIDI_PROBE_PIN_1`**
`[qdprobe]` is missing or failed to load. Both `[smart_effector]` and `[qdprobe]` must be
present. Neither works alone.

**`Option 'position_endstop' in section 'stepper_z' must be specified`**
Uncomment `position_endstop: -0.23` in `[stepper_z]`.

**`File contains parsing errors`, every line listed**
Your editor indented the pasted config. Klipper requires option lines to start at column
zero. Only the lines inside `activate_gcode` and `deactivate_gcode` are indented.

**The mesh comes out around -1.5 at every point**
`z_offset` has a real value instead of `0.000001`, so the mesh is corrected twice. Set
both `z_offset` values back to `0.000001`.

**The nozzle never reaches the plate during `get_zoffset`**
The pin is deployed when it should be stowed. Check `probe_deploy` and `probe_stow`
directions, and confirm `[output_pin probe_enable]` has `value: 1`.

**`PROBE_CALIBRATE` returns a negative number**
Something below the nozzle touched first: a deployed pin, or a blob of plastic. You do
not need `PROBE_CALIBRATE` with this setup. The piezo replaces it.

**`sudo systemctl` is not permitted**
QIDI blocks `systemctl` in the sudoers file. Restart services through Moonraker instead,
at **Machine > Services** in Fluidd. Moonraker can restart `klipper`, `moonraker`,
`klipper_mcu`, and `webcamd`.

## Reference

Working configuration from the test machine is in `config-backups/`. `gcode_macro-CHANGES.md`
records every change against the stock files, with the reason for each.
