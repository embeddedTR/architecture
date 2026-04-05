# Layered Architecture: Gömülü Sistemlerde Temiz ve Ölçeklenebilir Tasarım

Gömülü sistemlerde projeler küçük başlar ancak zamanla büyür:

- Yeni sensörler eklenir  
- Haberleşme modülleri eklenir  
- UI / kontrol mekanizmaları gelişir  

Eğer yazılım mimarisi doğru kurulmazsa sistem kısa sürede kontrol edilemez hale gelir.

İşte bu noktada **layered architecture (katmanlı mimari)** devreye girer.

---

## Layered Architecture Nedir?

Layered architecture, sistemi **katmanlara bölerek** her katmanın belirli bir sorumluluğu üstlendiği bir tasarım yaklaşımıdır.

Amaç:

- Bağımlılığı azaltmak  
- Kodun okunabilirliğini artırmak  
- Sistemi modüler hale getirmek  

---

## Temel Katmanlar

Gömülü sistemlerde yaygın olarak kullanılan yapı:

### 1. Application Layer (Uygulama Katmanı)

Sistemin “ne yaptığı” bu katmanda tanımlanır.

Örnek:

- State machine  
- İş mantığı (business logic)  
- Sensör karar mekanizmaları  

Bu katman donanım detaylarını **bilmemelidir**.

---

### 2. Middleware / Service Layer (Opsiyonel)

Application ile driver arasında köprü görevi görür.

Örnek:

- Haberleşme protokolleri  
- Veri işleme  
- Filtreleme algoritmaları  

---

### 3. Driver Layer (Sürücü Katmanı)

Donanım ile doğrudan iletişim kuran katmandır.

Örnek:

- GPIO driver  
- UART driver  
- ADC driver  
- RF modül driver  

Bu katman register seviyesinde çalışabilir.

---

### 4. Hardware Layer (Donanım Katmanı)

Fiziksel donanımın kendisidir.

- Mikrodenetleyici  
- Sensörler  
- RF modüller  
- PCB  

---

## Basit Bir Örnek

Yanlış yaklaşım:

```c
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0))
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET);
}

```

Bu kodda:

Application → HAL’a bağımlı
Donanım bağımlılığı yüksek

Doğru yaklaşım (katmanlı):
```c
if (button_is_pressed())
{
    led_turn_on();
}
```

Driver katmanı:

```c
int button_is_pressed(void)
{
    return HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);
}

void led_turn_on(void)
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET);
}
```

Application artık donanımdan bağımsızdır.
