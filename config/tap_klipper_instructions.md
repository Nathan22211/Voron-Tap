# Updating your Klipper config for Tap

There are a few changes you'll need to make in order to get Tap working properly.

1. Update your Z endstop: 
   
   - Under the `[stepper_z]` block, you'll want to comment out your `position_endstop` and change your `endstop_pin` so that it uses the virtual Z endstop for Tap.
   
      - Example: `endstop_pin: probe:z_virtual_endstop`
   
   - Be sure to *also* remove any automatically saved `[stepper_z] position_endstop: ...` value that may be found at the **bottom** of your `printer.cfg`file.

2. Update your Z homing position: 
   
   - If you use `[safe_z_home]`, change the location to the center of the bed. If you have a `[homing_override]` make sure that it moves the toolhead to the center of the bed before calling `G28 Z`.
   
3. Update your probe's offsets: 
   
   - Remember, with Tap, your nozzle IS the probe, so your `[probe] x_offset` and `[probe] y_offset` values should be 0 now. You'll need to manually calibrate the probe's Z offset by using `PROBE_CALIBRATE`.
   
4. Add Tap's `activate_gcode:` and `deactivate_gcode` 
   
   - This G-code will allow you to probe cold, but will also prevent you from probing with a nozzle at printing temperature (to try to preserve your build surface), and too fast, particularly on the 2.4, to prevent inaccurate readings, preemptive triggering of the probe, or; in the case of the trident; damaging the tool head due to high Z accelerations. This goes in the `[probe]` section of your config.  

```jinja

activate_gcode:
   {% set PROBE_TEMP = 150 %}
   {% set MAX_TEMP = PROBE_TEMP + 5 %}
   {% set ACTUAL_TEMP = printer.extruder.temperature %}
   {% set TARGET_TEMP = printer.extruder.target %}
   SET_KINEMATICS_LIMIT Z_ACCEL=15 # may need to be tuned per printer. 
   {% if TARGET_TEMP > PROBE_TEMP %}
      { action_respond_info('Extruder temperature target of %.1fC is too high, lowering to %.1fC' % (TARGET_TEMP, PROBE_TEMP)) }
      M109 S{ PROBE_TEMP }
   {% else %}
   # The temperature target is already low enough, but the nozzle may still be too hot.
      {% if ACTUAL_TEMP > MAX_TEMP %}
         { action_respond_info('Extruder temperature %.1fC is still too high, waiting until below %.1fC' % (ACTUAL_TEMP, MAX_TEMP)) }
         TEMPERATURE_WAIT SENSOR=extruder MAXIMUM={ MAX_TEMP }
      {% endif %}
   {% endif %}
on_deactivate_gcode:
   SET_KINEMATICS_LIMIT Z_ACCEL=<regular_accel> Z_VELOCITY=<regular_velocity>    
```

5. turn off `deactivate_on_each_sample`
   - if left on it will move away from the bed at regular Z-axis speed, which will cause inaccuracies in bed meshing. Add `deactivate_on_each_sample: False` to the probe section of your config. 
