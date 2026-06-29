# VL53L1X Driver — no_std Rust for ESP32

A minimal VL53L1X Time-of-Flight distance sensor driver based on https://github.com/pololu/vl53l1x-arduino library written in `no_std` Rust for ESP32 using Embassy + esp-hal.

---

## Hardware

| VL53L1X Pin | ESP32 Pin       |
| ----------- | --------------- |
| SDA         | GPIO21          |
| SCL         | GPIO22          |
| VCC         | 3.3V            |
| GND         | GND             |
| XSHUT       | GPIO (any free) |

Default I2C address: `0x29`

---

## Dependencies

```toml
[dependencies]
esp-hal = { version = "=1.0.0-beta.0", features = ["esp32","unstable"] }
esp-hal-embassy = "0.7.0"
embassy-executor = "..."
embassy-time = "..."
```

---

## Usage

```rust
// Initialize I2C and create the driver
let vl_i2c = I2c::new(peripherals.I2C0, Config::default()
    .with_frequency(Rate::from_khz(400)))
    .unwrap()
    .with_sda(peripherals.GPIO21)
    .with_scl(peripherals.GPIO22)
    .into_async();

let mut vl53l1x = VL53L1X::new(vl_i2c, 0x29).await;
vl53l1x.init().await; // writes default config and starts ranging

// Set distance mode (Short / Medium / Long)
vl53l1x.set_distance_mode(DistanceMode::Long).await;

// Set measurement timing budget (microseconds)
vl53l1x.set_measurement_timing_budget(50_000).await;

// Start continuous ranging
vl53l1x.start_continuous(0).await;

// Read distance (mm) — blocking until data is ready
let distance_mm = vl53l1x.read_range_single_milimeters(true).await;
info!("{}mm", distance_mm);

// Check range status
let status = vl53l1x.ranging_data.range_status;
info!("{}", vl53l1x.range_status_to_string(status).await);
```

---

## Distance Modes

| Mode    | Max Range (typical) | Notes                                      |
|---------|---------------------|--------------------------------------------|
| Short   | ~1.3 m              | Better ambient immunity                    |
| Long    | ~4.0 m              | Default; more susceptible to ambient light |

Call `set_distance_mode()` **before** `start_continuous()` to take effect.

---

## Range Status Values

The driver exposes a `RangeStatus` enum on every reading. Common values:

| Status                       | Meaning                                              |
|------------------------------|------------------------------------------------------|
| `RangeValid`                 | Measurement is good                                  |
| `SigmaFail`                  | Sigma (std dev) exceeds internal threshold           |
| `SignalFail`                 | Signal too weak                                      |
| `OutOfBoundsFail`            | Phase out of bounds — try a longer distance mode     |
| `HardwareFail`               | VCSEL or hardware error                              |
| `WrapTargetFail`             | Wrapped target, non-matching phases                  |
| `MinRangeFail`               | Target below minimum detection threshold             |

Always check `range_status` before trusting `range_mm`.

---

## ROI (Region of Interest)

You can restrict the sensor's field of view to a sub-region of the 16×16 SPAD array:

```rust
// Set ROI size (max 16x16)
vl53l1x.set_roi_size(8, 8).await;

// Set ROI center SPAD (default is 199 — center of array)
vl53l1x.set_roi_center(199).await;
```

> **Note on ROI center:** The VL53L1X lens **inverts** the image. To shift the FOV toward the upper-left physically, set the center SPAD toward the lower-right of the SPAD map. See the ST UM2555 user manual for the full SPAD grid diagram.

---

## Default Configuration

| Parameter              | Value        |
|------------------------|--------------|
| Distance Mode          | Long         |
| Timing Budget          | 50 ms        |
| Inter-measurement      | 0 (back-to-back) |
| I2C Frequency          | 400 kHz      |
| Default I2C Address    | `0x29`       |

---

## Timeout Handling

```rust
// Set I/O timeout in milliseconds (0 = disabled)
vl53l1x.set_timeout(500).await;

// Check if a timeout occurred since last call
if vl53l1x.time_out_accured().await {
    info!("Ranging timed out!");
}
```
