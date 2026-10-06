# Phase Balancing Installation Guide

This guide covers installation and configuration of the Smart Battery Pilot fork with phase balancing functionality.

## What's New in This Fork

This enhanced version adds **phase load balancing** for three-phase electrical systems:

### Key Features
- **Real-time phase monitoring**: Uses Tibber Pulse or similar sensors to monitor phase loads
- **Automatic power adjustment**: Dynamically adjusts battery charging power to prevent phase overload
- **Flexible monitoring modes**: Support for per-phase or overall system monitoring
- **Configurable safety margins**: Set your own safety margins for conservative operation
- **Seamless integration**: Works alongside existing price optimization features

### Use Case
Perfect for homes with:
- Three-phase electrical systems
- Battery connected to a single phase
- Need to avoid overloading specific phases during high-power charging
- Dynamic electricity tariffs (Tibber, EPEX Spot, etc.)

## Installation

### Method 1: HACS (Recommended)

1. **Add Custom Repository**
   - Open HACS in Home Assistant
   - Go to Integrations → ⋮ → Custom repositories
   - Add: `https://github.com/vhardono/ha-smart-battery-pilot`
   - Category: Integration

2. **Install Integration**
   - Search for "Smart Battery Pilot" in HACS
   - Click Install
   - Restart Home Assistant when prompted

3. **Configure Integration**
   - Go to Settings → Devices & Services → Add Integration
   - Search for "Smart Battery Pilot"
   - Follow the configuration wizard (includes new phase balancing step)

### Method 2: Manual Installation

1. **Download the Integration**
   ```bash
   cd /path/to/homeassistant/config
   mkdir -p custom_components
   cd custom_components
   git clone https://github.com/vhardono/ha-smart-battery-pilot.git smart_battery_pilot
   ```

2. **Restart Home Assistant**
   - Restart Home Assistant to load the new integration

3. **Configure via UI**
   - Settings → Devices & Services → Add Integration → Smart Battery Pilot

## Configuration

### New Phase Balancing Step

The configuration wizard now includes a 6th step for phase balancing:

#### Step 6: Phase Balancing Configuration

| Field | Description | Example Value |
|-------|-------------|---------------|
| Enable phase balancing | Turn feature on/off | `True` |
| Phase monitoring mode | "per_phase" or "overall" | `per_phase` |
| Phase selection | Which phase to monitor (L1/L2/L3/all) | `L2` |
| Phase current L1 | Tibber Pulse L1 current sensor | `sensor.tibber_pulse_current_l1` |
| Phase current L2 | Tibber Pulse L2 current sensor | `sensor.tibber_pulse_current_l2` |
| Phase current L3 | Tibber Pulse L3 current sensor | `sensor.tibber_pulse_current_l3` |
| Phase voltage L1 | Tibber Pulse L1 voltage sensor | `sensor.tibber_pulse_voltage_l1` |
| Phase voltage L2 | Tibber Pulse L2 voltage sensor | `sensor.tibber_pulse_voltage_l2` |
| Phase voltage L3 | Tibber Pulse L3 voltage sensor | `sensor.tibber_pulse_voltage_l3` |
| Max current per phase | Maximum current per phase (Amperes) | `25` |
| Phase voltage | Standard phase voltage (Volts) | `230` |
| Safety margin | Safety margin percentage | `10` |

### Example Configuration for User's Setup

For a Marstek Venus E on Phase B (L2) with Tibber Pulse:

```yaml
Enable phase balancing: True
Phase monitoring mode: per_phase
Phase selection: L2
Phase current L2: sensor.tibber_pulse_vrije_vogellaan_25_current_l2
Phase voltage L2: sensor.tibber_pulse_vrije_vogellaan_25_voltage_phase2
Max current per phase: 25
Phase voltage: 230
Safety margin: 10
```

## Verification

### 1. Check Integration Status
- Go to Settings → Devices & Services → Smart Battery Pilot
- Verify the integration is loaded and running
- Check for any configuration errors

### 2. Monitor Phase Balancing Logs
- Check Home Assistant logs for phase balancing messages:
  ```
  Phase balancing: reduced charge power from 2500 W to 1800 W (phase load: 15.2 A, available: 1850 W)
  ```

### 3. Verify Sensor Readings
- Developer Tools → States
- Check your Tibber Pulse phase sensors are updating
- Verify current and voltage values are reasonable

### 4. Test with Dry Run
- Enable dry run mode in the integration options
- Monitor logs to see planned power adjustments
- Verify phase balancing logic is working correctly

## Troubleshooting

### Phase Balancing Not Working

**Symptoms**: No phase balancing messages in logs, power not adjusted

**Solutions**:
1. Verify phase balancing is enabled in configuration
2. Check that Tibber Pulse sensors are available and updating
3. Verify phase selection matches your physical connection
4. Check logs for "Error calculating phase-balanced power" messages
5. Ensure sensor entity IDs are correct

### Sensors Not Available

**Symptoms**: Configuration shows "entity not found" errors

**Solutions**:
1. Verify Tibber integration is working
2. Check sensor entity IDs in Developer Tools → States
3. Ensure Tibber Pulse is properly configured
4. Restart Home Assistant after adding sensors

### Power Not Adjusting

**Symptoms**: Battery charges at full power despite high phase load

**Solutions**:
1. Check safety margin isn't set too high
2. Verify max current per phase is correct for your electrical system
3. Monitor actual phase current during charging
4. Check if phase load exceeds calculated limits
5. Review logs for calculation details

### Integration Won't Load

**Symptoms**: Integration fails to start or shows configuration errors

**Solutions**:
1. Verify all required configuration fields are filled
2. Check Home Assistant logs for specific error messages
3. Ensure you're using a compatible Home Assistant version
4. Try removing and re-adding the integration
5. Check for conflicts with other integrations

## Advanced Configuration

### Adjusting Safety Margin

For more conservative operation:
- Increase safety margin to 15-20%
- This leaves more headroom for unexpected loads

For more aggressive operation:
- Decrease safety margin to 5%
- This maximizes charging power but reduces safety buffer

### Overall Monitoring Mode

Use "overall" monitoring mode when:
- You want to balance across all three phases
- Your battery draws from multiple phases
- You have total system load concerns

Configuration:
```
Phase monitoring mode: overall
Phase selection: all
```

### Custom Sensor Integration

You can use non-Tibber sensors if they provide:
- Real-time current measurements per phase
- Voltage measurements per phase
- Regular updates (at least every minute)

Simply substitute your sensor entity IDs in the configuration.

## Performance Considerations

### Sensor Update Rate
- Phase balancing works best with sensors updating every 30-60 seconds
- Tibber Pulse typically updates every minute
- Slower updates may cause delayed power adjustments

### Calculation Overhead
- Phase balancing calculations are lightweight
- Minimal impact on Home Assistant performance
- Calculations only occur during charge slots

### Fallback Behavior
- If sensors become unavailable, falls back to planned power
- Logs warnings when sensor data is missing
- Continues normal price optimization

## Compatibility

### Home Assistant Version
- Requires Home Assistant 2024.x or later
- Tested on 2024.9 and 2025.x versions

### Integrations
- Works with any Marstek integration that supports script control
- Compatible with Tibber, EPEX Spot, Nordpool price integrations
- Vendor-neutral: works with any battery controllable via scripts

### Electrical Systems
- Designed for three-phase systems (230V per phase)
- Can be adapted for single-phase systems
- Supports various current ratings (16A, 25A, 35A, 40A, etc.)

## Support and Contributing

### Getting Help
- Check the main Smart Battery Pilot documentation
- Review the Marstek example configuration
- Check Home Assistant logs for detailed error messages
- Open an issue on GitHub for bugs or feature requests

### Contributing
- Fork the repository
- Create a feature branch
- Make your changes
- Submit a pull request
- Include documentation for new features

## Version Information

- **Base Version**: Smart Battery Pilot by cygnusb
- **Fork Version**: Phase Balancing Enhancement by vhardano
- **Status**: Stable, in active use
- **Last Updated**: 2026-09-30

## Future Enhancements

Planned improvements for future versions:
- Support for additional sensor types
- More sophisticated load prediction
- Integration with energy dashboard
- Advanced scheduling options
- Multi-battery coordination