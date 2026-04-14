# Callback Pattern

Gömülü sistemlerde katmanlar arasında iletişim kurarken sıkça karşılaşılan bir sorun vardır: alt katman (driver), üst katmana (uygulama) bir şey olduğunu nasıl haber verir?

Callback pattern bu sorunun en temiz çözümüdür.

---

## 🔧 Sorun: Ters Bağımlılık

```c
// KÖTÜ: Driver, uygulama fonksiyonunu doğrudan çağırıyor
// uart_driver.c
#include "app.h"  // driver, uygulamayı include etmek zorunda kalıyor

void UART_IRQHandler(void) {
    uint8_t byte = UART->DR;
    app_process_uart_byte(byte);  // sıkı bağımlılık
}
```

Bu durumda driver, uygulamadan habersiz olamaz. Başka bir projede yeniden kullanmak için kodu değiştirmek gerekir.

---

## ✅ Çözüm: Callback ile Gevşek Bağlantı

Driver, "bir şey olduğunda şu fonksiyonu çağır" diye talimat alır. Kimin fonksiyonu olduğunu bilmez.

### 1. Callback tipini tanımla

```c
// uart_driver.h
#ifndef UART_DRIVER_H
#define UART_DRIVER_H

#include <stdint.h>

// callback tipi: byte aldığında çağrılacak fonksiyon
typedef void (*uart_rx_callback_t)(uint8_t byte);

void uart_init(uint32_t baudrate);
void uart_register_rx_callback(uart_rx_callback_t cb);
void uart_send(const uint8_t *data, uint16_t len);

#endif
```

### 2. Driver içinde callback'i sakla ve çağır

```c
// uart_driver.c
#include "uart_driver.h"

static uart_rx_callback_t rx_callback = NULL;

void uart_register_rx_callback(uart_rx_callback_t cb) {
    rx_callback = cb;
}

void USART1_IRQHandler(void) {
    if (USART1->SR & USART_SR_RXNE) {
        uint8_t byte = USART1->DR;
        if (rx_callback != NULL) {
            rx_callback(byte);  // uygulamaya haber ver
        }
    }
}
```

### 3. Uygulama callback'i kaydeder

```c
// app.c
#include "uart_driver.h"

static void on_uart_byte_received(uint8_t byte) {
    // gelen byte'ı işle
    command_parser_feed(byte);
}

void app_init(void) {
    uart_init(115200);
    uart_register_rx_callback(on_uart_byte_received);
}
```

---

## 🔄 Çoklu Callback

Birden fazla olayı ayrı callback'lerle yönetebilirsin:

```c
// sensor_driver.h
typedef void (*sensor_data_ready_cb)(float temperature, float humidity);
typedef void (*sensor_error_cb)(uint8_t error_code);

typedef struct {
    sensor_data_ready_cb on_data_ready;
    sensor_error_cb      on_error;
} SensorCallbacks;

void sensor_register_callbacks(const SensorCallbacks *cbs);
```

```c
// app.c
static void on_sensor_data(float temp, float hum) {
    display_update(temp, hum);
}

static void on_sensor_error(uint8_t code) {
    log_error(MODULE_SENSOR, code);
}

void app_init(void) {
    SensorCallbacks cbs = {
        .on_data_ready = on_sensor_data,
        .on_error      = on_sensor_error,
    };
    sensor_register_callbacks(&cbs);
}
```

---

## ⚠️ ISR'da Callback Kullanırken Dikkat

Callback ISR içinden çağrılıyorsa:

```c
// ISR içinde çağrılan callback kısa ve hızlı olmalı
static void on_uart_byte_received(uint8_t byte) {
    // EVET: buffer'a yaz
    ring_buffer_push(&rx_buf, byte);

    // HAYIR: uzun işlem yapma
    // parse_full_command(byte);  // tehlikeli
    // HAL_Delay(10);             // kesinlikle yapma
}
```

ISR'dan gelen callback'lerde sadece flag set et veya buffer'a yaz, asıl işlemi main loop veya RTOS task'ında yap.

---

## 📌 Özet

| | Doğrudan Çağrı | Callback |
|---|---|---|
| Bağımlılık | Driver → Uygulama | Yok |
| Yeniden kullanım | Zor | Kolay |
| Test edilebilirlik | Zor | Kolay (mock callback) |
| Esneklik | Düşük | Yüksek |

Callback pattern, driver ve uygulama katmanını birbirinden bağımsız tutar. Driver "ne olduğunu" bildirir, "ne yapılacağını" uygulamaya bırakır.
