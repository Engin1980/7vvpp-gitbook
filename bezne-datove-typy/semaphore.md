# Semaphore

Máme např. **3 parkovací místa** a více aut (vláken), která se je snaží využít.

#### Princip

* `Semaphore(3)` → maximálně 3 vlákna mohou současně pokračovat
* `acquire()` → čekej, dokud není volné místo
* `release()` → uvolni místo pro další

```java
import java.util.concurrent.Semaphore;

public class ParkingExample {

    // Semaphore s 3 povoleními (parkovací místa)
    private static final Semaphore parking = new Semaphore(3);

    public static void main(String[] args) {
        for (int i = 1; i <= 10; i++) {
            int carId = i;
            new Thread(() -> park(carId)).start();
        }
    }

    private static void park(int carId) {
        try {
            System.out.println("Auto " + carId + " čeká na parkovací místo...");
            parking.acquire(); // zabere místo

            System.out.println("Auto " + carId + " parkuje.");
            Thread.sleep(2000); // simulace parkování

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            System.out.println("Auto " + carId + " odjíždí.");
            parking.release(); // uvolní místo
        }
    }
}
```

### Co je tady důležité

#### 1. Semaphore ≠ lock

* `synchronized` / `ReentrantLock` → **1 vlákno**
* `Semaphore(n)` → **až n vláken současně**

#### 2. `release()` musí být ve `finally`

Jinak při výjimce:

* „ztratíš“ povolení
* program se může **navždy zablokovat**

#### 3. Typické použití v reálných systémech

* databázový connection pool
* omezení počtu paralelních requestů
* throttling
* řízení přístupu k HW zdrojům
