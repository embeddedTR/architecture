# State Machine Nedir? Gömülü Sistemlerde Neden Bu Kadar Önemlidir?

Gömülü sistemlerde yazılım büyüdükçe, kodun kontrolü zorlaşır. Basit bir LED yakma uygulaması bile zamanla buton, sensör, haberleşme, hata yönetimi ve güç modları eklenince karmaşık hale gelir.

İşte bu noktada **state machine (durum makinesi)** yaklaşımı, sistemi düzenli, anlaşılır ve genişletilebilir tutmak için en güçlü yöntemlerden biridir.

---

## State Machine Nedir?

State machine, bir sistemin belirli **durumlar (states)** içinde çalışmasını ve belirli **olaylara (events)** göre bir durumdan başka bir duruma geçmesini tanımlar.

Basitçe:

- Sistem her an bir durumdadır
- Bir olay gerçekleşir
- Gerekirse başka bir duruma geçilir
- Yeni durumda farklı davranış gösterilir

---

## Temel Kavramlar

### 1. State (Durum)

Sistemin o anda bulunduğu çalışma modudur.

Örnek:

- `IDLE`
- `RUNNING`
- `ERROR`
- `STANDBY`

---

### 2. Event (Olay)

Sistemde değişime sebep olan tetikleyicidir.

Örnek:

- Butona basılması
- Sensör tetiklemesi
- Timeout oluşması
- Haberleşme paketi gelmesi

---

### 3. Transition (Geçiş)

Bir event sonucunda bir durumdan başka bir duruma geçilmesidir.

Örnek:

- `IDLE` -> `RUNNING`
- `RUNNING` -> `ERROR`

---

### 4. Action (Aksiyon)

Geçiş sırasında veya durum içinde yapılan işlemdir.

Örnek:

- LED yakmak
- RF paketi göndermek
- Motoru durdurmak
- Hata loglamak

---

## Basit Bir Örnek

Bir cihaz düşünelim:

- Başlangıçta bekleme modunda
- Butona basılınca çalışmaya başlıyor
- Süre dolunca tekrar beklemeye dönüyor

Bu sistemin state machine yapısı şöyle olabilir:

- `IDLE`
- `ACTIVE`

Event'ler:

- `BUTTON_PRESSED`
- `TIMEOUT`

Geçişler:

- `IDLE + BUTTON_PRESSED -> ACTIVE`
- `ACTIVE + TIMEOUT -> IDLE`

---

## Neden State Machine Kullanılır?

### 1. Kod daha düzenli olur

Kod, dağınık `if-else` blokları yerine net bir yapıya oturur.

### 2. Sistem davranışı anlaşılır hale gelir

Sistem hangi durumda ne yapıyor sorusunun cevabı nettir.

### 3. Yeni özellik eklemek kolaylaşır

Yeni bir durum veya olay eklemek tüm sistemi bozmaz.

### 4. Hata ayıklama kolaylaşır

Sorunun hangi durumda oluştuğunu görmek daha kolaydır.

### 5. Gömülü sistemlere çok uygundur

Özellikle event-driven çalışan yapılarda state machine çok doğal bir çözümdür.

---

## State Machine Olmadan Ne Olur?

State machine kullanılmazsa sistem genelde şu hale gelir:

- Her yerde `if`
- Her yerde bayrak değişkenleri
- Kimin neyi kontrol ettiği belli değil
- Yeni özellik ekleyince eski yapı bozuluyor

Örnek kötü yaklaşım:

```c
if (button_pressed)
{
    led_on();
    system_active = 1;
}

if (timeout && system_active)
{
    led_off();
    system_active = 0;
}

if (error_detected)
{
    led_blink();
    buzzer_on();
}

```
State machine örneği:

```c
typedef enum
{
    STATE_IDLE,
    STATE_ACTIVE
} state_t;

typedef enum
{
    EVENT_NONE,
    EVENT_BUTTON,
    EVENT_TIMEOUT
} event_t;

state_t current_state = STATE_IDLE;

void state_machine_run(event_t event)
{
    switch (current_state)
    {
        case STATE_IDLE:
            if (event == EVENT_BUTTON)
            {
                current_state = STATE_ACTIVE;
            }
            break;

        case STATE_ACTIVE:
            if (event == EVENT_TIMEOUT)
            {
                current_state = STATE_IDLE;
            }
            break;

        default:
            current_state = STATE_IDLE;
            break;
    }
}

