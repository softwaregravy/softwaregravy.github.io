---
layout: post
title: A Unique ID for ESP32
date: 2022-10-15 12:00:00 -0400
---

I've recently been looking for a unique id for my ESP32 development.
[I'm](https://stackoverflow.com/questions/73442663/esp32-arduino-ide-how-to-get-unique-id)
[not](https://stackoverflow.com/questions/70974797/how-to-get-arduino-esp32-device-details-in-code)
[alone](https://www.reddit.com/r/esp8266/comments/8xs1fq/creating_random_unique_ids_for_devices_using/).

The solution is to use the [MAC address](https://en.wikipedia.org/wiki/MAC_address).
I'm not going to go into details on why, but espressif is guaranteeing their chips have unique MAC addresses.
Sources: [1](https://docs.espressif.com/projects/esp-idf/en/release-v3.0/api-reference/system/base_mac_address.html),
[2](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/misc_system_api.html),
[3](https://www.esp32.com/viewtopic.php?t=17907).

But what actually comes out of the APIs and how do you rely on this?

Here are the outputs of a quick test program which I am using for reference.

```
Getting Mac Address. esp_efuse with uint64: 158581228646888
 as hex: 91CD31E8
Getting Mac Address. esp_efuse with uint8[6]: 2324920514558144
 as hex: E8 31 CD 91 3A 90
Getting Mac Address. esp_efuse with uint8[8]: 232492051455814400
 as hex: E8 31 CD 91 3A 90 0 0
Getting Mac Address. ESP libary: CHIP MAC: 158581228646888
 as hex: 91CD31E8
Getting Mac Address. WiFi Library: E8:31:CD:91:3A:90
```

And here is the program for reference:

```cpp
  Serial.print("Getting Mac Address. esp_efuse with uint64: ");
  uint64_t raw_esp_efuse_int64 = 0LL;
  esp_efuse_mac_get_default((uint8_t*)(&raw_esp_efuse_int64));
  Serial.println(String(raw_esp_efuse_int64));
  Serial.printf(" as hex: %X\n", raw_esp_efuse_int64);

  Serial.print("Getting Mac Address. esp_efuse with uint8[6]: ");
  uint8_t raw_esp_efuse_int_arr_6[6];
  esp_efuse_mac_get_default(raw_esp_efuse_int_arr_6);
  for (int i = 0; i < 6; i++) {
    Serial.print(String(raw_esp_efuse_int_arr_6[i]));
  }
  Serial.println("");
  Serial.printf(" as hex: %X %X %X %X %X %X\n", raw_esp_efuse_int_arr_6[0], raw_esp_efuse_int_arr_6[1],
                raw_esp_efuse_int_arr_6[2], raw_esp_efuse_int_arr_6[3], raw_esp_efuse_int_arr_6[4],
                raw_esp_efuse_int_arr_6[5]);

  Serial.print("Getting Mac Address. esp_efuse with uint8[8]: ");
  uint8_t raw_esp_efuse_int_arr_8[8];
  esp_efuse_mac_get_default(raw_esp_efuse_int_arr_8);
  for (int i = 0; i < 8; i++) {
    Serial.print(String(raw_esp_efuse_int_arr_8[i]));
  }
  Serial.println("");
  Serial.printf(" as hex: %X %X %X %X %X %X %X %X\n", raw_esp_efuse_int_arr_8[0], raw_esp_efuse_int_arr_8[1],
                raw_esp_efuse_int_arr_8[2], raw_esp_efuse_int_arr_8[3], raw_esp_efuse_int_arr_8[4],
                raw_esp_efuse_int_arr_8[5], raw_esp_efuse_int_arr_8[6], raw_esp_efuse_int_arr_8[7]);

  Serial.print("Getting Mac Address. ESP libary: ");
  uint64_t lib_esp_efuse = ESP.getEfuseMac();
  Serial.printf("CHIP MAC: %lld\n", lib_esp_efuse);
  Serial.println("");
  Serial.printf(" as hex: %X\n", lib_esp_efuse);

  Serial.print("Getting Mac Address. WiFi Library: ");
  Serial.println(WiFi.macAddress());
```

# Conclusion

I plan to use `WiFi.macAddress()` if I am using Arduino, or I would use the `esp_efuse_mac_get_default` with an array of 6.
This will be unique enough for my purposes, built only on espressif.

If I were building a much larger system and much more paranoid, I might prefix (or postfix) with something like my device version.
(Where a device version is just a number I assign while compiling my firmware that would only increment with runs of the hardware. Maybe stored in NVM.)
