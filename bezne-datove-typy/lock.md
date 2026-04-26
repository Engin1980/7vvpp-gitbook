# Lock

## Představení

...

## Příklad

...

## Implementace příkladu přes 'synchronized'

```java
import java.util.HashMap;
import java.util.Map;

public class ConfigStore {

    private final Map<String, String> config = new HashMap<>();

    /**
     * Jednoduché synchronizované čtení.
     * POZOR: žádný timeout, žádný fallback na úrovni locku.
     */
    public synchronized String getOrDefault(String key, String defaultValue) {
        return config.getOrDefault(key, defaultValue);
    }

    /**
     * Exkluzivní reload konfigurace
     */
    public synchronized void reload(Map<String, String> newConfig) {
        // simulace pomalého reloadu
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        config.clear();
        config.putAll(newConfig);
    }
}

```

### Co to **splňuje**

✅ **správnost**

* data jsou konzistentní
* žádné race conditions

✅ **jednoduchost**

* minimum kódu
* snadno čitelné

***

### Co to **NEUMÍ** (a to je zásadní)

#### 1️⃣ ŽÁDNÝ timeout

Když probíhá `reload()`:

Plain TextThread A: reload (1 s)\
Thread B: get() → čeká\
Thread C: get() → čeká\
Thread D: get() → čeká\
...\
\`\`\
Zobrazit více řádků

➡️ **čekají všichni** ➡️ **neomezeně dlouho** ➡️ **žádná kontrola latence**

Tohle:

* je OK v učebnici
* ❌ **špatně v serverech / backendu**

***

#### 2️⃣ ŽÁDNÝ fallback

Se `synchronized` **nelze říct**:

> „když to nejde teď hned, vrať default“

Thread:

* buď:
  * projde
* nebo:
  * čeká nekonečně dlouho

➡️ žádná degradace služby\
➡️ risk thread starvation

***

#### 3️⃣ Všechno je seriální

I čtení:

Javasynchronized get(...)\
\`\`\
Zobrazit více řádků

* čtenář blokuje čtenáře
* nulová paralelizace
* špatná škálovatelnost

## Implementace příkladu přes ReentrantLock

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;

public class ConfigStore {

    private final Map<String, String> config = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock(true);

    /**
     * Získá hodnotu nebo vrátí default,
     * pokud se nepodaří získat lock do timeoutMs.
     */
    public String getOrDefault(String key, String defaultValue, long timeoutMs) {
        boolean locked = false;
        try {
            locked = lock.tryLock(timeoutMs, java.util.concurrent.TimeUnit.MILLISECONDS);
            if (!locked) {
                return defaultValue;
            }
            return config.getOrDefault(key, defaultValue);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return defaultValue;
        } finally {
            if (locked) {
                lock.unlock();
            }
        }
    }

    /**
     * Exkluzivní reload konfigurace
     */
    public void reload(Map<String, String> newConfig) {
        lock.lock();
        try {
            // simulace pomalého reloadu
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            config.clear();
            config.putAll(newConfig);
        } finally {
            lock.unlock();
        }
    }
}
```

### Proč je to **architektonicky správně**

1️⃣ `tryLock(timeout)` = řízení latency

Javalock.tryLock(timeout, unit)\
Zobrazit více řádků

* **nikdy nečekáš donekonečna**
* thread:
  * buď rychle čte
  * nebo se rozhodne „kašlu na config“

To je **zásadní vlastnost v serverech**.

***

#### 2️⃣ Graceful degradation

Když probíhá reload:

* některé requesty:
  * dostanou default
  * systém pořád běží
* lepší než:
  * thread starvation
  * cascading timeouty

***

#### 3️⃣ Tohle `synchronized` neumí

Se `synchronized`:

Javasynchronized String get(...) {\
// čekáš navždy\
}\
Zobrazit více řádků

* ❌ žádný timeout
* ❌ žádná alternativa
* ❌ žádná kontrola latence

➡️ **nevhodné pro produkci**

***

### Chování v běhu (reálně)

```
Thread A: reload config (1s)
Thread B: getOrDefault → timeout 50 ms → default
Thread C: getOrDefault → timeout 50 ms → default
Thread D: getOrDefault → získá lock po reloadu
```

➡️ aplikace **nepřestane odpovídat**

***

### Důležité poznámky

#### ✅ Fair lock

Javanew ReentrantLock(true)\
Zobrazit více řádků

* lepší predikovatelnost
* žádné hladovění
* vhodné pro config / infra věci

(Mírně horší throughput – akceptovatelné)

***

#### ⚠️ Co by šlo zlepšit (vědomě neřešeno)

* read‑heavy scénář by byl lepší s:
  * `StampedLock` (optimistic read)
* ale:
  * tady **chceme timeout**
  * `StampedLock` neumí timeout pro optimistic read

## Implementace příkladu přes StampedLock

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.StampedLock;

public class ConfigStoreStamped {

    private final Map<String, String> config = new HashMap<>();
    private final StampedLock lock = new StampedLock();

    /**
     * Optimistic read:
     * - pokud někdo zapisoval, vrací default
     * - žádné blokující čekání
     */
    public String getOrDefault(String key, String defaultValue) {
        long stamp = lock.tryOptimisticRead();

        String value = config.get(key);

        // ověření, zda během čtení neproběhl zápis
        if (!lock.validate(stamp)) {
            return defaultValue;
        }

        return value != null ? value : defaultValue;
    }

    /**
     * Exkluzivní reload konfigurace
     */
    public void reload(Map<String, String> newConfig) {
        long stamp = lock.writeLock();
        try {
            // simulace drahého reloadu
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            config.clear();
            config.putAll(newConfig);
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

### Co je tady důležité (a proč je to **smysluplné**)

#### 1️⃣ `tryOptimisticRead()` **neblokuje**

Javalong stamp = lock.tryOptimisticRead();\
\`\`\
Zobrazit více řádků

* žádný mutex
* žádný wait
* jen čtení paměti

➡️ **ideální pro read‑heavy scénáře**

***

#### 2️⃣ `validate(stamp)` = kontrola konzistence

Javaif (!lock.validate(stamp)) {\
return defaultValue;\
}\
Zobrazit více řádků

* pokud během čtení:
  * někdo získal **write lock**
* stamp se zneplatní
* data **mohla být nekonzistentní**

➡️ **okamžitý fallback**

***

#### 3️⃣ Žádné blokování = žádné timeouty

To je **zásadní rozdíl** oproti `ReentrantLock`.

* u `StampedLock`:
  * **nečekáš**
  * **nerozhoduje čas**
  * rozhoduje **konflikt**

To je:

* extrémně rychlé
* ideální pro vysoký throughput

## Implementace příkladu přes ReadWriteLock

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ConfigStore {

    private final Map<String, String> config = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock(true);

    /**
     * Pokusí se přečíst hodnotu.
     * Pokud se nepodaří získat read lock do timeoutMs,
     * vrací defaultValue.
     */
    public String getOrDefault(String key, String defaultValue, long timeoutMs) {
        boolean locked = false;
        try {
            locked = lock.readLock().tryLock(timeoutMs, TimeUnit.MILLISECONDS);
            if (!locked) {
                return defaultValue;
            }
            return config.getOrDefault(key, defaultValue);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return defaultValue;
        } finally {
            if (locked) {
                lock.readLock().unlock();
            }
        }
    }

    /**
     * Exkluzivní reload konfigurace
     */
    public void reload(Map<String, String> newConfig) {
        lock.writeLock().lock();
        try {
            // simulace pomalého reloadu
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            config.clear();
            config.putAll(newConfig);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

### Proč je to **smysluplné** řešení

#### 1️⃣ Paralelní čtení

* více readerů může:
  * vstoupit **současně**
  * bez vzájemného blokování
* writer:
  * blokuje nové čtení
  * počká, až staré doběhnou

➡️ mnohem lepší než jeden `synchronized` mutex

***

#### 2️⃣ Timeout na čtení

Javalock.readLock().tryLock(timeoutMs, MILLISECONDS)\
Zobrazit více řádků

* **kontrola latence**
* žádný thread:
  * nečeká nekonečně
* při problému:
  * vrátíš default
  * systém zůstane responsivní

✅ **tohle `synchronized` neumí**

***

#### 3️⃣ Graceful degradation

Reálný běh:

```
Thread A: reload (drží write lock 1s)
Thread B: getOrDefault(timeout=50ms) → default
Thread C: getOrDefault(timeout=50ms) → default
Thread D: getOrDefault(timeout=50ms) → default
```

➡️ aplikace pořád odpovídá\
➡️ žádné hromadění threadů

### Kdy je `ReadWriteLock` správná volba

✅ když:

* máš **read‑heavy** data
* chceš **timeout**
* potřebuješ **konzistentní čtení**
* nechceš optimistiku (`StampedLock`)

❌ když:

* čtení je krátké a vzácné
* nebo je zápis častý
* nebo ti nevadí blokování

## Příklad - shrnutí

### Shrnutí: `synchronized` vs `ReentrantLock` vs `ReadWriteLock` vs `StampedLock`

| Vlastnost / Nástroj          | `synchronized`            | `ReentrantLock`    | `ReadWriteLock` | `StampedLock`  |
| ---------------------------- | ------------------------- | ------------------ | --------------- | -------------- |
| Základní typ                 | monitor (JVM)             | explicitní lock    | RW lock         | stamped lock   |
| Reentrantní                  | ✅                         | ✅                  | ✅               | ❌              |
| Timeout při zamykání         | ❌                         | ✅ (`tryLock`)      | ✅ (`tryLock`)   | ❌              |
| Try‑lock (neblokující pokus) | ❌                         | ✅                  | ✅               | ✅ (optimistic) |
| Přerušitelné čekání          | ❌                         | ✅                  | ✅               | ❌              |
| Paralelní čtení              | ❌                         | ❌                  | ✅               | ✅✅             |
| Exkluzivní zápis             | ✅                         | ✅                  | ✅               | ✅              |
| Optimistické čtení           | ❌                         | ❌                  | ❌               | ✅              |
| Neblokující čtení            | ❌                         | ❌                  | ❌               | ✅              |
| Fair scheduling              | ❌                         | ✅ (volitelně)      | ✅ (volitelně)   | ❌              |
| Složitost použití            | ✅✅✅                       | ✅✅                 | ✅               | ❌              |
| Riziko zneužití              | nízké                     | střední            | střední         | vysoké         |
| Typická latence              | vysoká                    | střední            | nízká           | velmi nízká    |
| Podpora downgrade/upgrade    | ❌                         | ❌                  | omezeně         | ✅              |
| Typické použití              | jednoduchá kritická sekce | timeout / fallback | read‑heavy data |                |

Významy řádků:

* **Základní typ** – jakého druhu synchronizační mechanismus je použit; určuje, jestli jde o implicitní jazykový konstruk­t, nebo explicitní objekt se stavem a API, a tím i jak moc má programátor kontrolu nad chováním synchronizace
* **Reentrantní** – zda může jedno vlákno získat stejný zámek víckrát po sobě, aniž by se samo zablokovalo; důležité pro strukturování kódu a volání metod „přes sebe“
* **Timeout při zamykání** – možnost omezit maximální dobu čekání na získání zámku; klíčové pro řízení latence a zabránění nekonečnému blokování vláken
* **Try‑lock (neblokující pokus)** – schopnost pokusit se zámek získat okamžitě a v případě neúspěchu pokračovat jinou cestou, místo čekání
* **Přerušitelné čekání** – zda čekání na zámek reaguje na přerušení vlákna (`interrupt`); důležité pro korektní rušení úloh a řízený shutdown
* **Paralelní čtení** – možnost, aby více vláken současně přistupovalo ke sdíleným datům pouze pro čtení, bez vzájemného blokování
* **Exkluzivní zápis** – vlastnost, že při zápisu do sdílených dat nemůže paralelně probíhat žádná jiná operace, ani čtení, aby byla zachována konzistence
* **Optimistické čtení** – čtení bez předchozího zamčení, spojené s následnou kontrolou, zda data nebyla mezitím změněna; předpokládá se, že ke konfliktům dochází vzácně
* **Neblokující čtení** – čtení, které nikdy nečeká na ostatní vlákna; buď proběhne ihned, nebo je výsledek odmítnut / degradován
* **Fair scheduling** – záruka, že vlákna získávají zámek v pořadí, v jakém o něj požádala; zvyšuje predikovatelnost, ale obvykle snižuje výkon
* **Složitost použití** – kolik pravidel a mentální režie musí programátor dodržovat, aby synchronizace byla správná a bezpečná
* **Riziko zneužití** – jak snadno lze mechanismus použít chybně tak, že vzniknou deadlocky, livelocky nebo nekonzistentní stav
* **Typická latence** – obvyklé chování z hlediska čekací doby vláken při souběhu; nejde o absolutní čísla, ale o charakteristický trend
* **Podpora downgrade / upgrade** – možnost přecházet mezi méně restriktivním a více restriktivním režimem synchronizace během jedné operace
* **Typické použití** – okruh problémů, pro které je daný synchronizační přístup přirozeně vhodný z hlediska výkonu, složitosti a správnos
