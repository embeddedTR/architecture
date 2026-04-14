# Embedded Sistemlerde Hata Yönetimi

Masaüstü yazılımda bir hata oluştuğunda uygulama kapanır, kullanıcı yeniden başlatır. Gömülü sistemlerde bu lüks çoğu zaman yoktur. Bir medikal cihaz, araç kontrol ünitesi veya endüstriyel sensör beklenmedik bir anda duramaz.

---

## ❌ Yaygın Yanlış Yaklaşımlar

### 1. Hataları görmezden gelmek

```c
// KÖTÜ
HAL_I2C_Master_Transmit(&hi2c1, addr, buf, len, 100);
// dönüş değeri kontrol edilmiyor
float temp = bme280_read();  // bu başarılı oldu mu?
```

### 2. Sonsuz döngüde donmak

```c
// KÖTÜ — sistemi tamamen kilitler
if (HAL_UART_Init(&huart1) != HAL_OK) {
    while (1);  // ne olduğunu asla bilemeyiz
}
```

### 3. Her yerde `printf` ile hata basmak

ISR içinde veya kritik zamanlama gerektiren yerlerde `printf` latency sorunlarına yol açar.

---

## ✅ Temel Prensipler

### 1. Her fonksiyon hata durumunu dönsün

```c
// hata kodları tanımla
typedef enum {
    ERR_OK       = 0,
    ERR_TIMEOUT  = 1,
    ERR_COMM     = 2,
    ERR_INVALID  = 3,
    ERR_HW_FAULT = 4,
} ErrorCode;

// fonksiyonlar ErrorCode dönsün
ErrorCode sensor_read(float *out_temp) {
    if (out_temp == NULL) return ERR_INVALID;

    if (i2c_transmit(SENSOR_ADDR, cmd, 1) != ERR_OK) {
        return ERR_COMM;
    }

    if (i2c_receive(SENSOR_ADDR, buf, 2) != ERR_OK) {
        return ERR_COMM;
    }

    *out_temp = parse_temp(buf);
    return ERR_OK;
}
```

### 2. Hata kodlarını çağıran yerde işle

```c
float temperature;
ErrorCode err = sensor_read(&temperature);

if (err != ERR_OK) {
    log_error(err);
    use_last_known_value();  // ya da güvenli bir varsayılan değer kullan
    return;
}

display_temperature(temperature);
```

---

## 🔁 Retry Mekanizması

Geçici iletişim hatalarında yeniden deneme mantığı ekle:

```c
#define MAX_RETRY 3

ErrorCode sensor_read_with_retry(float *out_temp) {
    for (int i = 0; i < MAX_RETRY; i++) {
        ErrorCode err = sensor_read(out_temp);
        if (err == ERR_OK) return ERR_OK;
        HAL_Delay(10);
    }
    return ERR_COMM;
}
```

---

## 📋 Hata Loglama

Gerçek zamanlı hata basmak yerine bir log buffer'ına yaz, uygun zamanda oku:

```c
#define LOG_SIZE 32

typedef struct {
    uint32_t  timestamp;
    ErrorCode code;
    uint8_t   module;
} LogEntry;

static LogEntry log_buf[LOG_SIZE];
static uint8_t  log_index = 0;

void log_error(uint8_t module, ErrorCode code) {
    log_buf[log_index % LOG_SIZE].timestamp = HAL_GetTick();
    log_buf[log_index % LOG_SIZE].code      = code;
    log_buf[log_index % LOG_SIZE].module    = module;
    log_index++;
}
```

---

## 🛡️ Güvenli Varsayılan Durum (Safe State)

Kritik sistemlerde kurtarılamaz bir hata olduğunda sistemi güvenli bir duruma al:

```c
void enter_safe_state(ErrorCode reason) {
    // tüm çıkışları kapat
    actuator_off_all();

    // hatayı kaydet (flash veya RTC backup register)
    fault_log_write(reason);

    // watchdog'u durdur ve sistem resetini bekle
    while (1) {
        // watchdog reset bekleniyor
    }
}
```

---

## 🐕 Watchdog Kullanımı

Yazılım kilitlenirse watchdog sistemi resetler. Sadece her şey yolundaysa besle:

```c
void main_loop(void) {
    while (1) {
        ErrorCode err = run_tasks();

        if (err == ERR_OK) {
            HAL_IWDG_Refresh(&hiwdg);  // watchdog'u besle
        } else {
            // hata varsa besleme — watchdog reset atar
            log_error(MODULE_MAIN, err);
        }
    }
}
```

---

## 📌 Özet

| Yaklaşım | Sonuç |
|---|---|
| Hataları görmezden gel | Sessiz arıza, tespit edilemez davranış |
| `while(1)` ile dondur | Sistem tamamen durur |
| Hata kodu döndür + işle | Kontrollü hata yönetimi |
| Retry + safe state + watchdog | Üretim kalitesi güvenilirlik |

Hata yönetimi sonradan eklenecek bir şey değil, mimarinin başından planlanması gereken bir unsurdur.
