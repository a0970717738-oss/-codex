# UWB + BLE Car Access Firmware Design

This document outlines the firmware architecture for a car access device that uses UWB presence detection to wake a microcontroller and perform authenticated unlocking over Bluetooth Low Energy (BLE).

## System Overview
- **Microcontroller (MCU):** Low-power MCU with sleep/stop mode and wake-up on GPIO/RTC.
- **UWB module:** Provides proximity detection (within ~2 m) and wake interrupt to MCU.
- **BLE radio:** Either integrated on MCU or discrete; supports extended advertising and GATT.
- **Secure element (optional):** Stores shared secret and performs HMAC to reduce key exposure.
- **Power domains:** Main MCU/BLE domain and always-on UWB presence domain.

## Functional Requirements
1. Remain in low-power sleep while no device is nearby.
2. UWB detects smartphone within ~2 m and raises wake interrupt.
3. MCU starts BLE advertising with a specific service UUID.
4. Smartphone app scans for advertisements, connects over GATT, and initiates challenge-response auth using shared secret.
5. If authentication succeeds, accept a GATT command to unlock the car door (drive vehicle-side actuator/ECU interface).
6. Return to advertising for a short grace period or go back to sleep when no connection is present.
7. Minimize power draw of UWB, BLE, and MCU; fail safe (do not unlock) on errors.

## Power Management
- **Base state:** MCU in STOP/STANDBY mode; UWB in low-power periodic ranging or presence detection mode. BLE radio off.
- **Wake sources:** UWB GPIO interrupt, watchdog, or timer for housekeeping.
- **Active window:** After wake, run advertising for a configurable window (e.g., 5–10 s). If no connection, stop advertising and return to sleep.
- **Connection timeout:** Disconnect and sleep if GATT session idle > configurable duration (e.g., 10 s).
- **Clocking:** Use low-frequency oscillator for sleep; switch to high-speed clock when active; disable unused peripherals (UART, ADC) when not needed.

## BLE Advertising & GATT
- **Advertising:** Non-connectable until wake; on UWB trigger, start connectable advertising at low duty cycle (e.g., 500–1000 ms interval) to balance discovery and power.
- **Advertisement payload:**
  - Flags: LE General Discoverable, BR/EDR not supported.
  - 128-bit Service UUID: `12345678-1234-5678-1234-56789abcdef0` (example).
  - Optional Tx power field for ranging calibration.
- **GATT Services:**
  - **Access Service (UUID above):**
    - `Challenge` characteristic (notify): MCU sends 16-byte random nonce.
    - `Response` characteristic (write): App writes HMAC-SHA256(nonce, shared_secret) truncated to 16 bytes.
    - `Control` characteristic (write): After auth, app writes `0x01` to request unlock.
    - `Status` characteristic (notify): Success/error codes.
- **Security:** Pairing not required; authentication is application-level. Consider LE Secure Connections pairing to protect against passive eavesdropping if acceptable to UX.

## Challenge–Response Flow
1. **UWB proximity** → MCU wake; start BLE advertising.
2. **App connects** to peripheral and discovers Access Service.
3. **MCU generates nonce** (16 bytes from TRNG) and sends via `Challenge` notify.
4. **App computes response** = `HMAC-SHA256(nonce, shared_secret)` → take first 16 bytes; writes to `Response` characteristic.
5. **MCU verifies** response; if match, set `authenticated = true` for session and send `Status: AUTH_OK` notify. Otherwise send `AUTH_FAIL` and optionally disconnect.
6. **App writes Control** `0x01` to request unlock. MCU checks `authenticated` and that request is within validity window (e.g., 5 s after auth). If valid, actuate unlock GPIO/CAN command and send `Status: UNLOCKED` notify.
7. **Timeouts:** Clear `authenticated` and disconnect on timeout or after handling command.

## UWB Proximity Handling
- Configure UWB module for periodic ranging/presence with threshold ~2 m. Use module interrupt to signal wake.
- After wake, optionally request a second confirmation to reduce false positives before enabling BLE.
- While BLE active, keep UWB in low-power listen or disable temporarily to reduce RF noise.

## Firmware State Machine
```text
[SLEEP]
  └─(UWB wake)→ [WAKE_INIT]
[WAKE_INIT]
  - Enable clocks/peripherals
  - Start BLE advertising
  - Start wake/adv timer
  └─→ [ADVERTISING]
[ADVERTISING]
  - Await connection
  - If timer expires and no connection → [SLEEP]
  └─(BLE connect)→ [GATT_SESSION]
[GATT_SESSION]
  - Exchange challenge/response
  - If auth success → set authenticated flag
  - On control write + authenticated → unlock + [UNLOCKED]
  - On timeout/disconnect → [SLEEP]
[UNLOCKED]
  - Pulse unlock GPIO/CAN message
  - Notify status
  - Disconnect and return to [SLEEP]
```

## Pseudocode Outline
```c
void main(void) {
    init_hw();
    enter_sleep();
}

void uwb_isr(void) {
    wake_event = UWB_WAKE;
}

void ble_evt_handler(event_t evt) {
    switch (evt.type) {
    case CONNECTED:
        session_reset();
        send_challenge();
        break;
    case WRITE_RESPONSE:
        if (verify_response(evt.data)) {
            session.authenticated = true;
            notify_status(AUTH_OK);
            auth_timestamp = now_ms();
        } else {
            notify_status(AUTH_FAIL);
            disconnect();
        }
        break;
    case WRITE_CONTROL:
        if (session.authenticated && (now_ms() - auth_timestamp < AUTH_WINDOW_MS)) {
            unlock_door();
            notify_status(UNLOCKED);
            disconnect();
        } else {
            notify_status(DENIED);
        }
        break;
    case DISCONNECTED:
        stop_ble();
        enter_sleep();
        break;
    }
}

void enter_sleep(void) {
    stop_ble();
    prepare_low_power();
    while (!wake_event) {
        __WFI(); // wait for interrupt (UWB)
    }
    wake_event = false;
    wake_init();
}

void wake_init(void) {
    enable_clocks();
    start_ble_adv();
    start_adv_timeout_timer();
}
```

## Security Considerations
- Store shared secret in secure element or MCU with flash read-out protection.
- Use TRNG for nonces; avoid predictable PRNGs.
- Throttle authentication attempts (e.g., disconnect after N failures, rate-limit wake triggers).
- Consider rolling keys or smartphone-bound keys (per-user provisioning) with key derivation.
- Protect against relay: combine UWB distance bounding and BLE auth window; optionally require RSSI/angle consistency.

## Power Optimization Tips
- Duty-cycle UWB presence detection; tune to minimal burst power that meets latency targets.
- Use long advertising intervals initially (e.g., 1000 ms) and shorten after the first scan response if permitted by timing requirements.
- Disable debug interfaces in production; clock gate crypto accelerators except during HMAC.
- Use connection parameters that minimize active radio time (e.g., 50–100 ms interval) while meeting UX latency.
- Enter deep sleep immediately after unlock or connection timeout.

## Provisioning & Testing Hooks
- UART/USB command to set shared secret and advertising parameters (disabled in production build).
- Manufacturing test mode triggered by pin strap to run RF and crypto self-tests.
