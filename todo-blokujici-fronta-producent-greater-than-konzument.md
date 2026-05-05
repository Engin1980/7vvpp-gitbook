# Blokující fronta - Producent -> Konzument

Fronta bez jakýchkoliv kontrol - naivní implementace:

{% code lineNumbers="true" %}
```java
import java.util.LinkedList;
import java.util.Queue;

public class RawQueue<T> {
    private final Queue<T> queue = new LinkedList<>();

    public void put(T item) {
        queue.add(item);
    }

    public T pop() {
        return queue.poll();
    }
}
```
{% endcode %}

Fronta bez omezení kapacity a bez synchronizace

{% code lineNumbers="true" %}
```java
import java.util.LinkedList;
import java.util.Queue;

public class BrokenQueue<T> {
    private final Queue<T> queue = new LinkedList<>();

    public void put(T item) {
        queue.add(item);
    }

    public T pop() {
        while (true) {
            if (!queue.isEmpty()) {
                return queue.poll();
            }

            try {
                Thread.sleep(10); 
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        }
    }
}
```
{% endcode %}

Fronta bez omezení kapacity bez wait/notify se sleepem:

{% code lineNumbers="true" %}
```java
import java.util.LinkedList;
import java.util.Queue;

public class DirtyQueue<T> {
    private final Queue<T> queue = new LinkedList<>();

    public synchronized void put(T item) {
        queue.add(item);
        // Žádné notify! Vlákno prostě položí balíček a odejde.
    }

    public T pop() {
        while (true) {
            synchronized (this) {
                if (!queue.isEmpty()) {
                    return queue.poll();
                }
            }

            // Tady je to "zprasení" - vlákno marně čeká a hádá, kdy se vrátit
            try {
                Thread.sleep(10); // Spinkej 10 ms a pak to zkus znova
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        }
    }
}
```
{% endcode %}

Fronta bez omezení kapacity se synchronizací a s wait/notify:

{% code lineNumbers="true" %}
```java
import java.util.LinkedList;
import java.util.Queue;

public class ThreadSafeQueue<T> {
    // Používáme LinkedList, protože efektivně implementuje frontu (FIFO)
    private final Queue<T> queue = new LinkedList<>();

    // Přidání prvku na konec fronty
    public synchronized void put(T item) {
        queue.add(item);
        // Notifikujeme jedno nebo všechna čekající vlákna, že přibyla data
        notifyAll(); 
    }

    // Vyjmutí prvku z hlavy fronty
    public synchronized T pop() {
        // Dokud je fronta prázdná, vlákno uvolní zámek a čeká
        while (queue.isEmpty()) {
            try {
                wait();
            } catch (InterruptedException e) {
                // Dobrá praxe: zachovat informaci o přerušení vlákna
                Thread.currentThread().interrupt();
                return null; 
            }
        }
        // Jakmile se probudí a fronta není prázdná, vyjme prvek
        return queue.poll();
    }

    // Pomocná metoda pro zjištění aktuální velikosti
    public synchronized int size() {
        return queue.size();
    }
}
```
{% endcode %}

Fronta s omezením kapacity a wait/notify:

{% code lineNumbers="true" %}
```java
import java.util.LinkedList;
import java.util.Queue;

public class ThreadSafeQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public ThreadSafeQueue(int capacity) {
        this.capacity = capacity;
    }

    // Metoda pro přidání prvku do fronty
    public synchronized void put(T item) {
        // Dokud je fronta plná, výrobce musí čekat
        while (queue.size() == capacity) {
            try {
                wait();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        queue.add(item);
        // Notifikujeme spotřebitele, že ve frontě přibyla data
        notifyAll();
    }

    // Metoda pro výběr prvku z fronty
    public synchronized T get() {
        // Dokud je fronta prázdná, spotřebitel musí čekat
        while (queue.isEmpty()) {
            try {
                wait();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        T item = queue.poll();
        // Notifikujeme výrobce, že se ve frontě uvolnilo místo
        notifyAll();
        return item;
    }
}
```
{% endcode %}
