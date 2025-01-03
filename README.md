# DS18B20 temperature sensor
A Rust [DS18B20](https://www.taydaelectronics.com/datasheets/A-072.pdf) temperature sensor driver for [embedded-hal](https://github.com/rust-embedded/embedded-hal) 1.0.0.

This device uses the 1-wire protocol, and requires using the [one-wire-bus](https://crates.io/crates/one-wire-bus)
library for the 1-wire bus.

## Quick Start

### Get Temperature
```rust
pub fn get_temperature<P, E>(
    delay: &mut impl DelayNs,
    tx: &mut impl Write,
    one_wire_bus: &mut OneWire<P>,
) -> OneWireResult<(), E>
    where
        P: OutputPin<Error=E> + InputPin<Error=E>,
        E: Debug
{
    // initiate a temperature measurement for all connected devices
    ds18b20_2::start_simultaneous_temp_measurement(one_wire_bus, delay)?;

    // wait until the measurement is done. This depends on the resolution you specified
    // If you don't know the resolution, you can obtain it from reading the sensor data,
    // or just wait the longest time, which is the 12-bit resolution (750ms)
    Resolution::Bits12.delay_for_measurement_time(delay);

    // iterate over all the devices, and report their temperature
    let mut search_state = None;
    loop {
        if let Some((device_address, state)) = one_wire_bus.device_search(search_state.as_ref(), false, delay)? {
            search_state = Some(state);
            if device_address.family_code() != ds18b20_2::FAMILY_CODE {
                // skip other devices
                continue;
            }
            // You will generally create the sensor once, and save it for later
            let sensor = Ds18b20::new(device_address)?;

            // contains the read temperature, as well as config info such as the resolution used
            let sensor_data = sensor.read_data(one_wire_bus, delay)?;
            //writeln!(tx, "Device at {:?} is {}°C", device_address, sensor_data.temperature);
            //Original above, patched below to fix unhandled result issue
            if let Err(e) = writeln!(tx, "Device at {:?} is {}°C", device_address, sensor_data.temperature) {
                println!("Writing error: {}", e.to_string());   
            }
        } else {
            break;
        }
    }
    Ok(())
}
```

### Configuration
```rust
pub fn test_config<P, E>(
    delay: &mut impl DelayNs,
    tx: &mut impl Write,
    one_wire_bus: &mut OneWire<P>,
) -> OneWireResult<(), E>
    where
        P: OutputPin<Error=E> + InputPin<Error=E>,
        E: Debug
{

    // Find the first device on the bus (assuming they are all Ds18b20's)
    if let Some(device_address) = one_wire_bus.devices(false, delay).next() {
        let device_address = device_address?;
        let device = Ds18b20::new(device_address)?;

        // read the initial config values (read from EEPROM by the device when it was first powered)
        let initial_data = device.read_data(one_wire_bus, delay)?;
        //writeln!(tx, "Initial data: {:?}", initial_data);
        //^^^ old version. possible solution below
        if let Err(e) = writeln!(tx, "Initial data: {:?}", initial_data) {
            println!("Writing error: {}", e.to_string());   
        }

        let resolution = initial_data.resolution;

        // set new alarm values, but keep the resolution the same
        device.set_config(18, 24, resolution, one_wire_bus, delay)?;

        // confirm the new config is now in the scratchpad memory
        let new_data = device.read_data(one_wire_bus, delay)?;
        //writeln!(tx, "New data: {:?}", new_data);
        //^^^ same problem with Err here.
        if let Err(e) = writeln!(tx, "New data: {:?}", new_data) {
            println!("Writing error: {}", e.to_string());   
        }

        // save the config to EEPROM to save it permanently
        device.save_to_eeprom(one_wire_bus, delay)?;

        // read the values from EEPROM back to the scratchpad to verify it was saved correctly
        device.recall_from_eeprom(one_wire_bus, delay)?;
        let eeprom_data = device.read_data(one_wire_bus, delay)?;
        //writeln!(tx, "EEPROM data: {:?}", eeprom_data);
        //^^^Err problem here too.
        if let Err(e) = writeln!(tx, "EEPROM data: {:?}", eeprom_data) {
            println!("Writing error: {}", e.to_string());   
        }

    }
    Ok(())
}
```
Example output
```
Initial data: SensorData { temperature: 85.0, resolution: Bits12, alarm_temp_low: 70, alarm_temp_high: 75 }
New data: SensorData { temperature: 85.0, resolution: Bits12, alarm_temp_low: 18, alarm_temp_high: 24 }
EEPROM data: SensorData { temperature: 85.0, resolution: Bits12, alarm_temp_low: 18, alarm_temp_high: 24 }
```
