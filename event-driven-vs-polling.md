# Event-Driven vs Polling: Gömülü Sistemlerde Doğru Yaklaşım

Gömülü sistemlerde yazılım mimarisini belirleyen en kritik kararlardan biri, sistemin **polling mi yoksa event-driven mı çalışacağıdır**.

Bu seçim, performans, güç tüketimi ve sistemin ölçeklenebilirliği üzerinde doğrudan etkilidir.

---

## 🔁 Polling Nedir?

Polling yaklaşımında CPU sürekli olarak belirli durumları kontrol eder.

```c
while(1)
{
    if(button_pressed())
    {
        handle_button();
    }
}
