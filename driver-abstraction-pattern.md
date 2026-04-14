# Driver Abstraction Pattern

Gömülü sistemlerde farklı donanımları (sensörler, ekranlar, iletişim modülleri) aynı arayüzle yönetebilmek büyük esneklik sağlar. Driver abstraction pattern tam olarak bunu sağlar.

---

## 🔧 Sorun: Her Sensör Farklı Kod

Projene iki farklı sıcaklık sensörü eklediğini düşün:

```c
// KÖTÜ: Her sensör için ayrı uygulama kodu
float temp;

if (sensor_type == SENSOR_DS18B20) {
    temp = ds18b20_read_temp();
} else if (sensor_type == SENSOR_BME280) {
    temp = bme280_get_temperature();
}
```

Üçüncü bir sensör eklendiğinde bu `if-else` zinciri büyümeye devam eder. Test etmek zorlaşır, bakım yükü artar.

---

## 🏗️ Çözüm: Ortak Arayüz (Driver Interface)

Tüm sensörler için ortak bir arayüz tanımla. Her sensör bu arayüzü kendi implementasyonuyla doldurur.

### 1. Ortak arayüzü tanımla

```c
// sensor.h
#ifndef SENSOR_H
#define SENSOR_H

#include <stdint.h>
#include <stdbool.h>

typedef struct {
    bool    (*init)(void);
    float   (*read_temperature)(void);
    float   (*read_humidity)(void);
    void    (*deinit)(void);
} SensorDriver;

#endif
```

### 2. DS18B20 implementasyonu

```c
// sensor_ds18b20.c
#include "sensor.h"
#include "ds18b20.h"

static bool ds18b20_drv_init(void) {
    return ds18b20_init() == DS18B20_OK;
}

static float ds18b20_drv_read_temp(void) {
    return ds18b20_get_temp_celsius();
}

static float ds18b20_drv_read_humidity(void) {
    return -1.0f;  // DS18B20 nem ölçmez
}

static void ds18b20_drv_deinit(void) {
    ds18b20_deinit();
}

const SensorDriver DS18B20_Driver = {
    .init             = ds18b20_drv_init,
    .read_temperature = ds18b20_drv_read_temp,
    .read_humidity    = ds18b20_drv_read_humidity,
    .deinit           = ds18b20_drv_deinit,
};
```

### 3. BME280 implementasyonu

```c
// sensor_bme280.c
#include "sensor.h"
#include "bme280.h"

static bool bme280_drv_init(void) {
    return bme280_init() == BME280_OK;
}

static float bme280_drv_read_temp(void) {
    return bme280_get_temperature();
}

static float bme280_drv_read_humidity(void) {
    return bme280_get_humidity();
}

static void bme280_drv_deinit(void) {
    bme280_power_off();
}

const SensorDriver BME280_Driver = {
    .init             = bme280_drv_init,
    .read_temperature = bme280_drv_read_temp,
    .read_humidity    = bme280_drv_read_humidity,
    .deinit           = bme280_drv_deinit,
};
```

### 4. Uygulama kodu hangi sensörün kullanıldığını bilmez

```c
// app.c
#include "sensor.h"

extern const SensorDriver BME280_Driver;

void app_run(void) {
    const SensorDriver *sensor = &BME280_Driver;

    if (!sensor->init()) {
        // hata yönetimi
        return;
    }

    float temp = sensor->read_temperature();
    float hum  = sensor->read_humidity();

    // yarın DS18B20'ye geçmek istersen:
    // const SensorDriver *sensor = &DS18B20_Driver;
    // uygulama kodu değişmez
}
```

---

## 🔄 Mock Driver ile Test

Bu pattern'in en büyük avantajlarından biri test edilebilirliktir. Gerçek donanım olmadan mock driver yazabilirsin:

```c
// sensor_mock.c — test ortamı için
static float mock_temp = 25.0f;

static bool mock_init(void)              { return true; }
static float mock_read_temp(void)        { return mock_temp; }
static float mock_read_humidity(void)    { return 60.0f; }
static void mock_deinit(void)            { }

const SensorDriver Mock_Driver = {
    .init             = mock_init,
    .read_temperature = mock_read_temp,
    .read_humidity    = mock_read_humidity,
    .deinit           = mock_deinit,
};
```

---

## 📌 Ne Zaman Kullanılmalı?

**Kullan:**
- Aynı türden birden fazla donanım seçeneği varsa
- Donanım değişebilir veya henüz belirlenmemişse
- Unit test yazmak istiyorsan

**Kullanma:**
- Yalnızca tek bir donanım kullanılacaksa ve değişmeyecekse
- Küçük, tek amaçlı bir projeyse — gereksiz karmaşıklık oluşturur
