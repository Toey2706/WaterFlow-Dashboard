# WaterFlow Dashboard

Application side of the ESP32-S3 water flow monitor by team **OneNight Miracle**: the ThingsBoard Cloud dashboard and the alert service that links the device to the Dormitory Maintenance Reporting System.

- Live dashboard (public, read only): https://thingsboard.cloud/dashboard/d93b85d0-e41c-11f1-b86a-c14319f12de9?publicId=495be850-e42c-11f1-b8a1-9db681a717b8
- Firmware: https://github.com/Toey2706/WaterFlow-Firmware
- Dormitory Maintenance Reporting System (where `water-alert` is deployed): https://github.com/Toey2706/Dorm-care

## How the parts work together

```
YF-S201 ──> ESP32-S3 ──MQTT──> ThingsBoard Cloud ──> dashboard (gauge, total, chart, reset button)
                │
                └──HTTPS (water keeps flowing for LEAK_SECONDS)──> Supabase function water-alert
                                                                     ├─> LINE card to residents bound to the room
                                                                     └─> Android push to dorm admins
```

The alert only informs people. It does not open a repair ticket: the resident decides and taps "แจ้งซ่อม" on the LINE card, which starts the normal reporting flow.

## Dashboard

`thingsboard/water_flow_monitor.json` is the dashboard exported from ThingsBoard Cloud. To use it in another account:

1. Create a device and copy its access token into the firmware `secrets.h`.
2. Dashboards > Add dashboard > Import dashboard, choose the JSON file.
3. Open each widget and select your device as the data source.

Telemetry keys sent by the firmware: `flow_lpm`, `total_l`, `flowing`, `rssi`, `flow_seconds`, `leak`.
The Reset total button calls RPC `resetTotal`. The firmware also accepts `testAlert`, which sends one alert right away for demonstrations.

For the public link to show data, the dashboard and the device both have to be in entity groups that are made public.

## water-alert

`water-alert/` is a copy of the Supabase Edge Function kept in the Dorm-care repository, where it shares the LINE and Firebase helpers. The source of truth is `supabase/functions/water-alert` there.

Request:

```
POST https://<project>.supabase.co/functions/v1/water-alert
x-device-key: <WATER_DEVICE_KEY>
Content-Type: application/json

{"room":"A1-01","minutes":10,"flow_lpm":3.2,"total_l":41.5}
```

Responses: `200` with how many LINE messages and Android pushes were sent, `401` for a wrong device key, `404` for an unknown room, `400` for invalid numbers.

Setup in the Supabase project:

1. Add the secret `WATER_DEVICE_KEY` (Edge Functions > Secrets). Use the same value as in the firmware `secrets.h`.
2. Deploy the function `water-alert` with JWT verification turned off (the device key is checked instead).

Tests run with Node from `supabase/functions` in Dorm-care: `node --test test/*.test.js`.

โค้ดฉบับล่าสุดของ water-alert, water-status และแท็บน้ำในแอปอยู่ที่ https://github.com/Toey2706/Dorm-care
