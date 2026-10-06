# Marstek Venus E with Phase Balancing

Short description of the setup: Marstek Venus E battery connected to Phase B (L2) of a three-phase system, controlled via Marstek Local API integration with Tibber Pulse for dynamic pricing and phase load monitoring.

## Architecture

```
Home Assistant
    ├── Smart Battery Pilot (Phase Balancing Fork)
    │   ├── Price optimization (Tibber integration)
    │   ├── Phase load balancing (Tibber Pulse sensors)
    │   └── Plan execution
    ├── Marstek Local API Integration
    │   └── Marstek Venus E battery (Phase B/L2)
    └── Tibber Integration
        └── Tibber Pulse (real-time consumption & phase monitoring)
```

The control path works through:
1. Smart Battery Pilot optimizes charge/discharge times based on EPEX spot prices
2. Phase balancing logic reads Tibber Pulse phase current/voltage sensors
3. Adjusts charging power to avoid overloading Phase B (L2)
4. Calls Marstek control scripts via Home Assistant

## Prerequisites

### Hardware
- Marstek Venus E battery (max charge: 2500W, max discharge: 800W)
- Tibber Pulse installed (real-time metering)
- Three-phase electrical system (230V per phase, 25A max per phase)
- Marstek connected to Phase B (L2)

### Home Assistant Integrations
- **Marstek Local API**: https://github.com/Flodesirat/ha-marstek-local-api
- **Tibber**: Official Home Assistant integration
- **Smart Battery Pilot (Phase Balancing Fork)**: This modified version

### Marstek Local API Setup
1. Enable Local API in Marstek app
2. Install Marstek Local API integration via HACS
3. Add your Marstek device to Home Assistant
4. Note the entity IDs for control

## Scripts

The four scripts Smart Battery Pilot calls. The charge and export scripts receive the variable
`power_w` which will be automatically adjusted for phase balancing.

### Marstek Control Scripts

```yaml
sbp_force_charge:
  alias: "SBP: Force charge (Marstek)"
  fields:
    power_w:
      description: Charge power in W (automatically adjusted for phase balancing)
  sequence:
    - service: marstek_local_api.set_mode
      data:
        device_id: "your_marstek_device_id"
        mode: "charge"
    - service: marstek_local_api.set_charge_power
      data:
        device_id: "your_marstek_device_id"
        power_w: "{{ power_w }}"
    - service: marstek_local_api.set_discharge_power
      data:
        device_id: "your_marstek_device_id"
        power_w: 0

sbp_block_discharge:
  alias: "SBP: Block discharge (idle) (Marstek)"
  sequence:
    - service: marstek_local_api.set_mode
      data:
        device_id: "your_marstek_device_id"
        mode: "auto"
    - service: marstek_local_api.set_charge_power
      data:
        device_id: "your_marstek_device_id"
        power_w: 0
    - service: marstek_local_api.set_discharge_power
      data:
        device_id: "your_marstek_device_id"
        power_w: 0

sbp_auto_mode:
  alias: "SBP: Auto mode (Marstek)"
  sequence:
    - service: marstek_local_api.set_mode
      data:
        device_id: "your_marstek_device_id"
        mode: "auto"

sbp_force_discharge:
  alias: "SBP: Force discharge to grid (Marstek)"
  fields:
    power_w:
      description: Discharge power in W
  sequence:
    - service: marstek_local_api.set_mode
      data:
        device_id: "your_marstek_device_id"
        mode: "discharge"
    - service: marstek_local_api.set_discharge_power
      data:
        device_id: "your_marstek_device_id"
        power_w: "{{ power_w }}"
    - service: marstek_local_api.set_charge_power
      data:
        device_id: "your_marstek_device_id"
        power_w: 0
```

**Note**: Replace `your_marstek_device_id` with your actual Marstek device ID from the integration.

## Integration Configuration

### Step 1: Price Source
─────────────────────┬──────────────
Config flow field    │Entity / value
─────────────────────┼──────────────
Price forecast entity│`sensor.tibber_electricity_price_today` (or your Tibber price sensor)
Price offset         │0.187 (adjust for your local grid fees/taxes)
Feed-in tariff       │0.08 (or your local feed-in tariff)
─────────────────────┴──────────────

### Step 2: Battery Parameters
─────────────────────┬──────────────
Config flow field    │Entity / value
─────────────────────┼──────────────
SOC entity           │`sensor.marstek_battery_soc` (from Marstek integration)
Capacity             │Your battery capacity in kWh (e.g., 10.0)
Max charge power     │2500 (W)
Max discharge power  │800 (W)
Min SOC              │10 (%)
Max SOC              │95 (%)
Efficiency           │90 (%)
─────────────────────┴──────────────

### Step 3: Control Scripts
─────────────────────┬──────────────
Config flow field    │Entity / value
─────────────────────┼──────────────
Charge script        │script.sbp_force_charge
Idle script          │script.sbp_block_discharge
Auto script          │script.sbp_auto_mode
Export script        │script.sbp_force_discharge (optional)
─────────────────────┴──────────────

### Step 4: Consumption
─────────────────────┬──────────────
Config flow field    │Entity / value
─────────────────────┼──────────────
Consumption sensor   │`sensor.tibber_pulse_consumption` (or your total consumption sensor)
Temperature entity   │(optional)
Has heat pump        │False (or True if applicable)
─────────────────────┴──────────────

### Step 5: Phase Balancing (NEW)
─────────────────────┬──────────────
Config flow field    │Entity / value
─────────────────────┼──────────────
Enable phase balancing│True
Phase monitoring mode│per_phase
Phase selection      │L2 (your Marstek is on Phase B)
Phase current L2     │`sensor.tibber_pulse_vrije_vogellaan_25_current_l2`
Phase voltage L2     │`sensor.tibber_pulse_vrije_vogellaan_25_voltage_phase2`
Max current per phase│25 (A)
Phase voltage        │230 (V)
Phase safety margin  │10 (%)
─────────────────────┴──────────────

### Step 6: Tuning
─────────────────────┬──────────────
Config flow field    │Entity / value
─────────────────────┼──────────────
Spread threshold     │0.20 (EUR/kWh)
Discharge mode       │self_consumption
Training days        │60
─────────────────────┴──────────────

## Verification

Monitor these sensors to confirm each action works:

### Phase Balancing Verification
- Check logs for "Phase balancing: reduced charge power from X W to Y W"
- Monitor Phase B current during charging: should not exceed 25A
- Verify power adjustment based on Tibber Pulse L2 current sensor

### Battery Control Verification
- `sensor.marstek_battery_soc` - should increase during charge slots
- `sensor.marstek_battery_power` - should show charge/discharge power
- `sensor.marstek_mode` - should reflect current mode (charge/auto/discharge)

### Price Optimization Verification
- Smart Battery Pilot sensors show current/next actions
- Battery charges during low-price hours
- Battery discharges during high-price hours

## Notes / Pitfalls

### Marstek Integration Specifics
- **API Rate Limits**: Marstek Local API may have rate limits - avoid rapid script calls
- **Device ID**: Ensure you use the correct device ID from the Marstek integration
- **Mode Changes**: Some Marstek firmware versions may have mode change delays

### Phase Balancing Specifics
- **Sensor Availability**: Ensure Tibber Pulse sensors are available and updating
- **Phase Selection**: Double-check your Marstek is connected to the correct phase
- **Safety Margin**: The 10% safety margin can be adjusted based on your comfort level
- **Total System Load**: This monitors only the battery phase, not total house load

### Tibber Integration
- **Price Updates**: Tibber prices are published around 13:00 CET for the next day
- **Sensor Accuracy**: Tibber Pulse provides real-time data, but may have slight delays
- **Entity Names**: Your Tibber entity names may differ from the examples

### General
- **Dry Run Mode**: Always test with dry_run=True first to verify behavior
- **Log Monitoring**: Check Home Assistant logs for phase balancing adjustments
- **Backup Plan**: Have manual control available via Marstek app as fallback

## Troubleshooting

### Phase Balancing Not Working
- Check that Tibber Pulse phase sensors are available: `sensor.tibber_pulse_*_current_l*`
- Verify phase selection matches your physical connection
- Check logs for "Error calculating phase-balanced power"
- Ensure phase balancing is enabled in configuration

### Battery Not Responding
- Verify Marstek Local API integration is working
- Check script syntax and device IDs
- Test scripts manually via Developer Tools → Services
- Check Marstek device status in the app

### Price Optimization Issues
- Verify Tibber price entity is updating correctly
- Check spread threshold isn't too high for your market
- Ensure battery SOC limits are reasonable
- Review training days and consumption data

## Advanced Configuration

### Custom Phase Balancing Parameters
For different electrical setups, you can adjust:
- **Max current per phase**: Set according to your main fuse (e.g., 35A, 40A)
- **Safety margin**: Increase for conservative operation (e.g., 15-20%)
- **Phase monitoring mode**: Use "overall" if you want total system balancing

### Multiple Batteries
If you have multiple batteries on different phases:
- Create separate Smart Battery Pilot instances for each
- Configure phase selection appropriately for each instance
- Coordinate via additional automations if needed

### Seasonal Adjustments
Consider adjusting parameters seasonally:
- Higher safety margin in winter (heating loads)
- Lower safety margin in summer (less overall load)
- Adjust spread threshold based on price volatility