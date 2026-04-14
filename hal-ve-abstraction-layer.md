# HAL ve Abstraction Layer Tasarımı

Gömülü sistemlerde en sık yapılan hatalardan biri, uygulama kodunu doğrudan donanıma bağlamaktır. Bu yaklaşım başlangıçta hızlı görünse de ilerleyen süreçte büyük bir bakım yükü oluşturur.

---

## 🔧 Sorun: Donanıma Sıkışmış Kod

```c
// KÖTÜ: Uygulama kodu doğrudan register'a yazıyor
void app_run(void) {
    GPIOA->ODR |= (1 << 5);   // LED yak
    HAL_Delay(500);
    GPIOA->ODR &= ~(1 << 5);  // LED söndür
}
```

Bu kodu farklı bir mikrodenetleyiciye taşımak istersen her satırı tek tek değiştirmek zorunda kalırsın. Test etmek de neredeyse imkansızdır.

---

## 🏗️ Çözüm: Abstraction Layer

Abstraction layer, donanıma özgü kodu bir soyutlama arkasına gizler. Üst katmanlar "ne yapılacağını" bilir, "nasıl yapılacağını" bilmez.

```
┌─────────────────────────────┐
│       Uygulama Katmanı      │  ← donanımdan habersiz
├─────────────────────────────┤
│     Abstraction Layer       │  ← arayüzü tanımlar
├─────────────────────────────┤
│       Driver Katmanı        │  ← donanıma özgü kod
├─────────────────────────────┤
│         Donanım             │
└─────────────────────────────┘
```

---

## ✅ Uygulama

### 1. Arayüzü tanımla (abstraction layer)

```c
// led.h — donanımdan bağımsız arayüz
#ifndef LED_H
#define LED_H

void led_init(void);
void led_on(void);
void led_off(void);
void led_toggle(void);

#endif
```

### 2. Donanıma özgü implementasyonu yaz (driver katmanı)

```c
// led_stm32.c — STM32'ye özgü implementasyon
#include "led.h"
#include "stm32f4xx_hal.h"

void led_init(void) {
    __HAL_RCC_GPIOA_CLK_ENABLE();
    GPIO_InitTypeDef cfg = {
        .Pin  = GPIO_PIN_5,
        .Mode = GPIO_MODE_OUTPUT_PP,
        .Pull = GPIO_NOPULL,
    };
    HAL_GPIO_Init(GPIOA, &cfg);
}

void led_on(void)     { HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);   }
void led_off(void)    { HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); }
void led_toggle(void) { HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);                }
```

### 3. Uygulama katmanı sadece arayüzü kullanır

```c
// app.c — donanımdan tamamen bağımsız
#include "led.h"

void app_run(void) {
    led_on();
    HAL_Delay(500);
    led_off();
}
```

---

## 🔄 Platforma Taşıma

Yarın ESP32'ye geçmek istersen sadece `led_esp32.c` yazarsın, uygulama kodu tek satır değişmez:

```c
// led_esp32.c
#include "led.h"
#include "driver/gpio.h"

#define LED_PIN 2

void led_init(void)   { gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT); }
void led_on(void)     { gpio_set_level(LED_PIN, 1); }
void led_off(void)    { gpio_set_level(LED_PIN, 0); }
void led_toggle(void) { gpio_set_level(LED_PIN, !gpio_get_level(LED_PIN)); }
```

---

## ⚠️ Dikkat Edilmesi Gerekenler

- Abstraction layer gereksiz yere karmaşık olmamalı. Tek LED için 5 dosya oluşturmak overkill olabilir.
- Soyutlama seviyesi projenin büyüklüğüne göre ayarlanmalı.
- Performans kritik yerlerde (ISR içi) abstraction overhead'i göz önünde bulundur.

---

## 📌 Özet

| | Doğrudan Donanım Erişimi | Abstraction Layer |
|---|---|---|
| Geliştirme hızı | Hızlı | Yavaş (başlangıç) |
| Taşınabilirlik | Yok | Yüksek |
| Test edilebilirlik | Zor | Kolay |
| Bakım maliyeti | Yüksek | Düşük |

Projen büyüdükçe abstraction layer'ın değeri katlanarak artar.
