# ToDo: Lock

## Motivace

Téma Lock představuje základní stavební kámen souběžnosti v Javě. Zatímco `synchronized` je vestavěný mechanismus jazyka, balíček `java.util.concurrent.locks` nabízí nástroje pro jemnější a profesionálnější řízení přístupu ke sdíleným datům.

V paralelním programování potřebujeme zajistit, aby se vlákna "nepoprala" o stejná data. Tradiční `synchronized` bloky jsou sice jednoduché, ale v produkčních systémech často narážejí na své limity – zejména nemožnost nastavit timeout nebo efektivně škálovat čtení. Dalším omezením je blokové pojetí `synchronized` - uzamčení i odemčení musí být v rámci jednoho bloku - nelze jej tedy "roztáhnout" mezi dvě funkce. Moderní Java zámky (Locks) tyto problémy řeší.

#### Nemožnost nastavit timeout

U klasického bloku `synchronized` platí pravidlo "všechno, nebo nic". Pokud vlákno narazí na zamčený monitor, vstoupí do stavu blokování a bude čekat neomezeně dlouho, dokud se zámek neuvolní. Programátor nemá žádný nástroj, jak toto čekání přerušit nebo omezit časem.

V produkčním prostředí to znamená obrovské riziko. Pokud jedno vlákno uvízne v dlouhé operaci (například kvůli pomalému I/O nebo chybě v logice) a drží zámek, všechna ostatní vlákna, která se pokusí o stejnou operaci, se "nastohují" za ním. To vede k vyčerpání vláknového fondu a úplnému zamrznutí aplikace. Moderní `Lock` tento problém řeší metodou `tryLock(time, unit)`, která umožní vláknu říct: „Zkus se zamknout, ale pokud se to nepovede do 100 milisekund, vzdej to a pokračuj alternativní cestou.“

#### Neefektivní škálování čtení

Klíčové slovo `synchronized` vytváří takzvaný exkluzivní zámek (mutex). To znamená, že v danou chvíli může být v kritické sekci pouze jedno jediné vlákno, bez ohledu na to, co tam dělá.

V mnoha aplikacích (jako je zmíněný `ConfigStore`) však vlákna data většinu času pouze čtou a nemění je. Z logického hlediska není důvod, proč by deset vláken nemohlo číst stejnou konfigurační hodnotu ve stejný okamžik. `synchronized` to však nepovolí – první čtenář zamkne dveře a druhý čtenář musí čekat venku, přestože by si navzájem do dat nezasahovali.

Tento jev se nazývá seriální úzké hrdlo. Výkon aplikace se nezvyšuje s počtem jader procesoru, protože všechna vlákna musí projít stejným bodem postupně za sebou. Nástroje jako `ReadWriteLock` umožňují tento problém vyřešit tím, že dovolí neomezenému počtu čtenářů pracovat paralelně, a exkluzivitu vynutí až v okamžiku, kdy se objeví zapisovatel.

#### Blokové omezení `synchronized`

Klíčové slovo `synchronized` je pevně svázáno s blokem kódu nebo tělem metody, což vytváří takzvanou strukturovanou synchronizaci; zámek se automaticky získá při vstupu do bloku a automaticky uvolní při jeho opuštění. To sice zvyšuje bezpečnost proti zapomenutému odemčení, ale znemožňuje to pokročilejší logiku, kdy potřebujeme zámek získat v jedné metodě a uvolnit jej až v metodě úplně jiné. Rozhraní `Lock` naproti tomu nabízí nestrukturované zamykání, které umožňuje rozdělit proces získání a uvolnění zámku do různých částí kódu. Díky tomu lze realizovat například řetězové zamykání (hand-over-hand locking) v datových strukturách typu spojový seznam, kde vlákno získá zámek pro další uzel dříve, než uvolní zámek uzlu předchozího, což je s blokovou strukturou `synchronized` technicky nerealizovatelné.

#### Shrnutí dopadů

Pokud systém neumí timeouty a škálování čtení:

1. Latence: I jednoduchý dotaz na konfiguraci může trvat sekundy, pokud právě probíhá zápis.
2. Propustnost: Server zvládne jen zlomek požadavků, protože procesor tráví většinu času přepínáním zablokovaných vláken místo reálné práce.
3. Stabilita: Systém je náchylný k řetězové havárii (cascading failure) – jedno pomalé vlákno postupně zastaví všechna ostatní.
4. Omezení při tvorbě kódu: Určité relativně jednoduché úlohy nelze vůbec realizovat, nebo je lze realizovat jen za pomocí obezliček komplikujících přehlednost a efektivnost kódu.

## Implementace

### Rozhraní Lock

Základním stavebním kamenem v Javě je rozhraní `java.util.concurrent.locks.Lock`. Reprezentuje abstrakci obecného zámku umožňujícího omezení přístupu ke zdroji.

Základní operace rozhraní `Lock` umožňují jemné řízení přístupu:

* `lock()`: Tato operace získá zámek, nebo zablokuje aktuální vlákno, dokud zámek není k dispozici. Je to nejbližší ekvivalent k `synchronized`.
* `unlock()`: Uvolní zámek. Pokud vlákno získalo zámek vícekrát (reentrance), musí jej stejný početkrát uvolnit, aby byl dostupný pro ostatní.
* `tryLock()`: Pokusí se získat zámek okamžitě. Pokud je volný, vrátí `true`, v opačném případě ihněď vrací `false`, čímž dává vláknu možnost vykonat náhradní logiku místo čekání.
* `tryLock(long time, TimeUnit unit)`: Pokročilá varianta, která čeká na získání zámku pouze po zadaný časový interval. Pokud se zámek nepodaří získat včas, operace se ukončí.
* `lockInterruptibly()`: Získá zámek, ale na rozdíl od standardního `lock()` umožní vláknu reagovat na signál přerušení (`interrupt`), což je zásadní pro korektní ukončování úloh.

### Třída ReentrantLock

Nejpoužívanější implementací tohoto rozhraní je třída `ReentrantLock`. Přívlastat „reentrantní“ znamená, že pokud vlákno již zámek drží, může jej opakovaně získat (například při rekurzivním volání nebo volání jiné metody téže třídy), aniž by se samo zablokovalo. `ReentrantLock` navíc umožňuje volitelný režim spravedlnosti (fairness), kde při nastavení na `true` garantuje, že zámek získá vlákno, které na něj čeká nejdéle. Na rozdíl od `synchronized` bloků se u explicitních zámků musí programátor starat o uvolnění, což se standardně provádí v bloku `finally`.&#x20;

Platí, že zámek musí uvolňovat to vlákno, které jej  uzamčelo. Navíc, pokud stejné vlákno uzamklo zámek několikrát (což může, protože _reentrant_), tak jej musí odemknout ve stejném počtu.

### Rozhraní ReadWriteLock

V systémech, kde operace čtení výrazně převažují nad zápisy, se standardní exkluzivní zámky stávají úzkým hrdlem, protože nutí čtenáře přistupovat k datům postupně jeden po druhém. Hlavní motivací pro zavedení `ReadWriteLock` je umožnit maximální paralelizaci čtení. Zatímco u běžného zámku (`ReentrantLock`) blokuje jeden čtenář všechny ostatní, `ReadWriteLock` vychází z logiky, že čtení dat je bezpečná operace, která nemění stav, a proto může probíhat v mnoha vláknech současně. Cílem je zvýšit propustnost aplikace tím, že se exkluzivita vynucuje pouze v okamžiku, kdy je potřeba data modifikovat, čímž se předchází zbytečnému čekání v situacích, kdy nedochází ke konfliktům.

Rozhraní `java.util.concurrent.locks.ReadWriteLock` definuje dvojici zámků: jeden pro čtení (`readLock()`) a druhý pro zápis (`writeLock()`). Nejpoužívanější implementací je `ReentrantReadWriteLock`. Tento mechanismus se řídí specifickými pravidly: zámek pro čtení může držet libovolný počet vláken najednou, pokud právě nikdo nezapisuje. Zámek pro zápis je však striktně exkluzivní – pokud vlákno zapisuje, žádné jiné vlákno nesmí číst ani zapisovat. Implementace navíc podporuje tzv. reentranci, takže vlákno držící zámek pro zápis může získat i zámek pro čtení (downgrade), ale opačný postup (upgrade z čtení na zápis) není přímo povolen, aby se předešlo deadlockům.

Klasickou implementací tohoto rozhraní je `ReentrantReadWriteLock`.

#### Základní operace

Na rozdíl od ostatních zámků, práce s tímto rozhraním vyžaduje volání specifických metod pro každý typ operace:

* `readLock().lock()`: Získá zámek pro čtení. Pokud jiná vlákna drží zámek pro čtení, operace proběhne okamžitě. Pokud však jiné vlákno právě zapisuje, čtenář se zablokuje.
* `readLock().unlock()`: Uvolní zámek pro čtení. Je nutné jej volat v bloku `finally` stejně jako u běžných zámků.
* `writeLock().lock()`: Získá exkluzivní zámek pro zápis. Vlákno musí počkat, dokud všechna vlákna (čtenáři i zapisovatelé) zámek neuvolní.
* `readLock().tryLock(long time, TimeUnit unit)`: Umožňuje pokusit se o čtení s časovým limitem, což je klíčové pro udržení responzivity systému při dlouhotrvajících zápisech.

### Třída StampedLock

Hlavní motivací pro vznik `StampedLock` byla snaha o dosažení maximálního výkonu prostřednictvím takzvaného optimistického čtení. Tento přístup vychází z předpokladu, že ke konfliktu mezi čtením a zápisem dochází v praxi jen zřídka. Namísto blokování ostatních vláken si čtenář pouze vyzvedne číselný lístek (razítko) a přečte data bez jakéhokoliv zamykání. Teprve po dokončení čtení ověří, zda je jeho razítko stále platné, což z něj činí ideální nástroj pro jemně laděné komponenty citlivé na latenci, kde by standardní zamykání představovalo zbytečnou brzdu.

Třída `java.util.concurrent.locks.StampedLock` se od ostatních zámků liší tím, že není reentrantní, což znamená, že vlákno nemůže získat stejný zámek opakovaně bez rizika uváznutí. Vnitřně nepracuje s objekty zámků, ale vrací hodnotu typu `long`, která slouží jako unikátní identifikátor stavu (razítko). Implementace nabízí tři režimy přístupu: exkluzivní zápis, pesimistické čtení (podobné `ReadWriteLock`) a unikátní optimistické čtení. Vývojář musí při použití optimistického režimu explicitně implementovat validaci razítka a v případě neúspěchu zvolit náhradní strategii, například opakování pokusu nebo přechod na těžší, pesimistický zámek.

## Příklad

Uvažujme systém, kde potřebujeme model/třídu `ConfigStore` realizující mechanismus **sdíleného úložiště konfigurace v paměti**, typicky používaný v serverové nebo paralelní aplikaci. Jeho hlavním účelem je umožnit **bezpečný a konzistentní přístup k datům z více vláken**.

Typické vlastnosti a požadavky na takovou komponentu jsou:

* konfigurace je **čtena velmi často** (např. při zpracování každého requestu),
* konfigurace je **měněna relativně zřídka** (reload, refresh, hot‑config),
* čtení musí být **rychlé a odolné vůči zablokování**,
* zápis musí být **atomický a konzistentní**,
* systém by měl ideálně umožnit **degradaci chování** (např. vrátit default, když konfigurace není dočasně dostupná),
* v produkčním prostředí je důležitá **predikovatelná latence** a vyhnutí se hromadění čekajících vláken.

`ConfigStore` je tedy zástupný příklad pro celou třídu problémů: _sdílený stav, hodně čtení, málo zápisů, více vláken, tlak na latenci a stabilitu_.

Následující sekce vysvětlují jednotlivé přístupy, které lze při řešení v Java využít.

### Implementace příkladu přes 'synchronized'

Přístup přes `synchronized` řeší souběžný přístup ke sdíleným datům tím, že serializuje všechny operace pomocí jednoho implicitního zámku, bez možnosti řídit čekání, latenci nebo fallback chování.

* Řeší základní problém **vzájemného vyloučení** při přístupu ke sdíleným datům.
* Zajišťuje korektnost a paměťovou viditelnost bez nutnosti dalšího kódu.
* Neřeší kontrolu čekání: pokud je zámek obsazen, vlákno **čeká neomezeně dlouho**.
* Neumožňuje rozlišit čtení a zápis ani reagovat na přetížení systému.
* Vhodné jako výchozí, demonstrační nebo velmi jednoduché řešení, nikoliv pro latency‑citlivé scénáře.

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

Výhody:

* správnost - data jsou konzistentní, nevznikají race conditions
* jednoduchost - minimum kódu, snadno čitelné

Nevýhody:

* žádné omezení doby čekání - pokud se bude dlouho čekat při updatu dat, všichni čtenáři čekají do dokončení operace -> thread starvation
* žádný fallback - buď se to zamkne, nebo se čeká, žádné podmíněné vyhodnocení -> degradace služby, thread startvation
* zbytečné blokování - čtenáři se blokuji navzájem, žádná paralelizace, špatná škálovatelnost

## Implementace příkladu přes ReentrantLock

Přístup pomocí `ReentrantLock` poskytuje explicitní řízení zámku, které umožňuje pokus o uzamčení, časové omezení čekání a přerušitelnost, čímž dává aplikaci kontrolu nad latencí a degradačním chováním.

* Řeší problém **nekontrolovaného čekání** zavedením explicitní práce se zámkem.
* Umožňuje nastavit **timeout**, zkusit zámek získat podmíněně a při neúspěchu použít fallback.
* Umožňuje **přerušitelné čekání**, což je důležité pro správné rušení úloh.
* Stále pracuje s jedním exkluzivním zámkem, takže **čtenáři se blokují navzájem**.
* Hodí se tam, kde je důležitá kontrola nad latencí, ale struktura dat je jednoduchá.

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

Výhody:

* timeout při nedostupnosti po stanovený interval
* po timeoutu možnost získat default hodnotu

Nevýhody:

* čtenáři se blokují navzájem a přitom se neovlivňují

## Implementace příkladu přes StampedLock

Přístup pomocí `StampedLock` umožňuje extrémně rychlé, neblokující čtení pomocí optimistického přístupu s následnou validací, za cenu vyšší složitosti a absence reentrance i timeoutu.

* Řeší problém **latence čtení způsobené zamykáním**.
* Umožňuje **optimistické (neblokující) čtení**, které proběhne bez zamykání a pouze se validuje.
* Čtení je extrémně rychlé a neblokuje se ani při probíhajícím zápisu.
* Přináší vyšší složitost, nereentranci a nutnost ruční validace.
* Hodí se tam, kde čtení výrazně převažuje nad zápisem a systém snese fallback při konfliktu.

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

Výhody:

* okamžité získání hodnoty (optimistic read)
* žádné blokování, žádné timeouty, nemůže vzniknout deadlock
* zachovaná konzistence dat

Nevýhody:

* při extrémně častých zápisech neustálé opakované čtení

## Implementace příkladu přes ReadWriteLock

Přístup pomocí `ReadWriteLock` odděluje čtení od zápisu tak, že umožňuje paralelní čtení více vlákny a současně zajišťuje exkluzivní, konzistentní zápis, čímž zlepšuje škálovatelnost u read‑heavy dat.

* Řeší problém **špatné škálovatelnosti při častém čtení**.
* Odděluje přístup pro čtení a zápis: více čtenářů může pracovat paralelně.
* Stále zachovává možnost **timeoutu a fallbacku**.
* Zápis je exkluzivní a blokuje další čtení, aby byla zachována konzistence.
* Vhodné pro read‑heavy struktury, kde potřebuješ konzistentní snapshot a kontrolu čekání.

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

Výhody:

* paralelní čtení, extrémně efektivní při řídkém zápisu
* konzistence
* timeout při čtení, náhrada výchozí hodnotou

Nevýhody:

* větší režie a obecně čtení pomalejší než `StampedLock`

## Příklad - shrnutí

Shrnutí: `synchronized` vs `ReentrantLock` vs `ReadWriteLock` vs `StampedLock`

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

## Bonus - Implementace přes CAS/neblokující

{% hint style="info" %}
Více pro info ohledně CAS viz [princip-cas-operace.md](../neblokujici-pristupy/princip-cas-operace.md "mention").
{% endhint %}

Přístup přes CAS (Compare‑And‑Swap) řeší synchronizaci tím, že atomicky vyměňuje celý stav bez zámků, takže čtení ani zápisy nikdy neblokují a systém má konstantní, predikovatelnou latenci.

* Řeší problém synchronizace **úplným odstraněním zámků**.
* Stav je reprezentován jako **neměnitelný objekt**, který je atomicky nahrazován.
* Čtení je vždy okamžité, bez čekání a bez rizika deadlocku.
* Zápis je atomická výměna reference; čtenáři nikdy nejsou blokováni.
* Nejvhodnější řešení pro konfiguraci a metadata, kde:
  * se zapisuje zřídka,
  * nevadí vytvoření nové instance dat,
  * je klíčová jednoduchost a predikovatelná latence.

```java
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

public class ConfigStore {

    // Atomická reference na immutable snapshot
    private final AtomicReference<Map<String, String>> config =
            new AtomicReference<>(Map.of());

    /**
     * Non-blocking čtení – vždy okamžité
     */
    public String getOrDefault(String key, String defaultValue) {
        return config.get().getOrDefault(key, defaultValue);
    }

    /**
     * Non-blocking reload přes CAS
     */
    public void reload(Map<String, String> newConfig) {
        // vytvoříme immutable snapshot
        Map<String, String> snapshot = Map.copyOf(newConfig);

        // atomicky vyměníme celý stav
        config.set(snapshot);
        // (CAS zde není cyklický, protože write nemá konfliktovou logiku)
    }
}
```

Výhody:

* neblokující přístup, nikdy se nečeká
* zachovaná konzistence
* rychlé, nepracuje se se zamykáním

Nevýhody:

* nepřehlednější kód

### CAS

CAS operace je reprezentována objektem `AtomicReference` a jehe metodou `set()`, která je atomická, tedy z pohledu JWM:

* garantovaná atomicita
* **happens‑before** kontrakt mezi zapisovatelem a čtenářem

{% hint style="info" %}
**Happens-before** kontrakt slouží jako záruka, že pokud operace A "předchází" operaci B, pak výsledek operace A musí být pro operaci B viditelný.

Hlavním problémem souběžnosti není jen to, že dvě vlákna mohou měnit data naráz, ale především to, že zápis provedený jedním vláknem nemusí být pro druhé vlákno okamžitě viditelný. Procesory mají vlastní vyrovnávací paměti (L1, L2, L3 cache), a pokud není vynucena synchronizace těchto dat, vlákno může pracovat s kopií, která neodpovídá hlavní paměti/stavu cache v jiného jádra procesoru.
{% endhint %}

Je tedy garantováno, že po provedení zápisu všechna vlákna vidí nově vložené hodnoty.

### Varianta s explicitním `compareAndSet`

Ve výše uvedeném příkladu, kdy nahrazujeme celý slovník, je tato varianta bezpečná i tehdy, když bude zápis provádět více zapisujících vláken, protože každé vlákno nahrazuje celý původní set za nový.

Pokud bychom však potřebovali držet konzistenci stavů a volání mezi objekty (např. zapisovatl by objekt načetl, provedl drobné změny s vytvořením nového objektu a ten by pak chtěl podstrčit za původní objekt), nebylo by garantováno, že mu někdo neprovedl další změny v původním objektu mezi jeho načtením a aktualizací dat. V takovém případě bychom potřebovali ještě ochranu garantující, že nahrazujeme takový objekt, který čekáme (ze kterého jsme původně vycházeli). Pro toto použití je funkce `compareAndSet()`.

```java
public void reload(Map<String, String> newConfig) {
    Map<String, String> snapshot = Map.copyOf(newConfig);

    while (true) {
        Map<String, String> current = config.get();
        // nějaké změny
        if (config.compareAndSet(current, snapshot)) {
            return;
        }
        // jiný writer mezitím změnil stav → retry
    }
}
```

Toto řešení je:

* stále **bez locků**
* retry jen při souběhu writerů
* readers pořád 100 % non‑blocking

Ve většině backendů je toto lepší než jakýkoliv přístup postavený nad zámky.

### Srovnání CAS s předchozími řešeními

<table><thead><tr><th width="233">Kritérium</th><th width="235">Locky</th><th width="216">CAS snapshot</th></tr></thead><tbody><tr><td>Blokování</td><td>ano</td><td>❌</td></tr><tr><td>Latence</td><td>proměnná</td><td>konstantní</td></tr><tr><td>Komplexita</td><td>vyšší</td><td>nižší</td></tr><tr><td>Read throughput</td><td>omezený</td><td>prakticky neomezený</td></tr><tr><td>Deadlock risk</td><td>ano</td><td>❌</td></tr></tbody></table>
