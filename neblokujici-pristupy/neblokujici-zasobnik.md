# Neblokující zásobník

Neblokující fronta je složitá (TODO)

Neblokující zásobník spojovým seznamem

{% code lineNumbers="true" %}
```java
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.LockSupport;

public class NonBlockingSet<T> {
    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }

    private final AtomicReference<Node<T>> head = new AtomicReference<>();

    public void put(T item) {
        Node<T> newNode = new Node<>(item);
        // "Swap" pomocí CAS: zkusíme přenastavit head na nový prvek
        while (true) {
            Node<T> currentHead = head.get();
            newNode.next = currentHead;
            if (head.compareAndSet(currentHead, newNode)) {
                return; // Povedlo se
            }
            // Pokud se nepovedlo (jiné vlákno nás předběhlo), smyčka jede znovu
        }
    }

    public T get() {
        while (true) {
            Node<T> currentHead = head.get();
            if (currentHead == null) {
                LockSupport.parkNanos(100); // Čekáme, až něco přibude
                continue;
            }

            // Zkusíme "odstřihnout" první prvek (výměna head za head.next)
            if (head.compareAndSet(currentHead, currentHead.next)) {
                return currentHead.value;
            }
        }
    }
}
```
{% endcode %}

#### \`AtomicReference\` vs obyčejná proměnná

Myšlenka: _„Zápis do reference (odkazu) je v Javě atomický sám o sobě, tak proč to řešit?“_ To je sice pravda, ale u vícevláknového programování jsou řešeny dva zásadní problémy právě pomocí `AtomicReference`:

**A. Viditelnost (Visibility)**

Moderní procesory jsou extrémně rychlé a disponují vlastními mezipamětmi (L1, L2, L3 cache). Pokud je v jednom vlákně změněna hodnota proměnné, může tato změna zůstat uložena v jeho lokální cache a ostatní jádra procesoru (na nichž běží jiná vlákna) mohou nadále vidět starou hodnotu.

* Pomocí `AtomicReference` (podobně jako při použití klíčového slova volatile) je vynuceno, aby každá změna byla okamžitě zapsána do hlavní paměti a aby byla ostatními vlákny z hlavní paměti znovu načtena.

**B. Operace „Přečti-Uprav-Zapiš“**

Jedná se o nejdůležitější důvod. Většina operací nespočívá pouze v zápisu, ale obsahuje logiku: „Změň hodnotu na `B`, ale pouze v případě, že je stále `A`“.

U běžné proměnné by bylo nutné provést následující kroky:

* Hodnota je přečtena.
* Je provedeno porovnání.
* Je zapsána nová hodnota.

Mezi prvním a třetím krokem však může dojít k přerušení vláknem operačním systémem a hodnota může být změněna jiným vláknem. Pomocí `AtomicReference` jsou tyto kroky provedeny jako jedna nedělitelná (atomická) instrukce přímo na procesoru.

***

## ABA problém

**ABA problém** představuje klasickou past v neblokujících algoritmech (CAS). Lze si představit scénář se zásobníkem (Stack), kde má být vyjmut prvek z vrcholu:

1. V zásobníku jsou prvky A → B → C.
2. Vláknem 1 je zahájena operace odebrání prvku `A`. Je přečteno, že na vrcholu je `A` a že dalším prvkem (novým vrcholem) bude `B`.
3. Následně je Vlákno 1 přerušeno (těsně před provedením compareAndSet — CAS).
4. Vlákno 2 následně provede:
   * Odebrání prvku `A`.
   * Odebrání prvku `B` (zásobník nyní obsahuje pouze `C`).
   * Opětovné vložení prvku `A` (může jít o stejný objekt nebo jiný se stejnou adresou).
5. Stav zásobníku je nyní `A` → `C`.
6. Po obnovení běhu Vlákna 1 je vyhodnoceno: „Na vrcholu je stále `A`, stav odpovídá očekávání, lze provést záměnu za `B`.“

Výsledek: Operace swap je úspěšně provedena a jako vrchol je nastaven prvek `B`. **Tento prvek však již v paměti neexistuje (nebo je neplatný), zatímco prvek `C` je nenávratně ztracen.**

Jak je tomu zabráněno?

V Javě je ABA problém nejčastěji řešen pomocí verzování:

* Je použita třída `AtomicStampedReference`, která uchovává nejen odkaz na objekt, ale i „razítko“ (číslo verze).
* I v případě, že se hodnota změní z `A` zpět na `A`, dojde ke zvýšení čísla verze (např. z 1 na 3), takže CAS operace ve Vláknu 1 selže, protože je očekávána verze 1.

{% code lineNumbers="true" %}
```java
import java.util.concurrent.atomic.AtomicStampedReference;
import java.util.concurrent.locks.LockSupport;

public class AbaSafeStack<T> {
    
    // Vnitřní uzel pro propojený seznam
    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }

    // Držíme referenci na hlavu + "razítko" (int stamp)
    // Počáteční stav: null, verze 0
    private final AtomicStampedReference<Node<T>> head = new AtomicStampedReference<>(null, 0);

    public void put(T item) {
        Node<T> newNode = new Node<>(item);
        int[] stampHolder = new int[1]; // Pomocné pole pro získání aktuálního razítka

        while (true) {
            Node<T> currentHead = head.get(stampHolder);
            int currentStamp = stampHolder[0];
            
            newNode.next = currentHead;
            
            // CAS: Změň head jen pokud se nezměnil objekt ANI razítko
            if (head.compareAndSet(currentHead, newNode, currentStamp, currentStamp + 1)) {
                return;
            }
            // Pokud selhalo, někdo jiný změnil head nebo verzi, zkusíme to znovu
        }
    }

    public T get() {
        int[] stampHolder = new int[1];

        while (true) {
            Node<T> currentHead = head.get(stampHolder);
            int currentStamp = stampHolder[0];

            if (currentHead == null) {
                // Čekání, dokud se něco neobjeví (tvůj požadavek)
                LockSupport.parkNanos(100); 
                continue;
            }

            Node<T> nextNode = currentHead.next;

            // CAS: Odstraňujeme hlavu a zvyšujeme razítko
            if (head.compareAndSet(currentHead, nextNode, currentStamp, currentStamp + 1)) {
                return currentHead.value;
            }
        }
    }
}
```
{% endcode %}

**Proč tato verze řeší ABA?**

Pojďme se vrátit k tomu příkladu se zásobníkem A -> B -> C:

1. Vlákno 1 si přečte hlavu `A` a razítko `10`.
2. Vlákno 2 mezi tím vyjme `A`, vyjme `B` a vrátí tam `A`. Protože ale pokaždé použilo `compareAndSet`, razítko se pokaždé zvýšilo. Teď je na vrcholu sice opět `A`, ale s razítkem `13`.
3. Vlákno 1 se probudí a zkusí svůj swap:
   * Objekt souhlasí? Ano (je tam `A`).
   * Razítko souhlasí? Ne (očekává `10`, ale v paměti je `13`).
4. CAS selže. Vlákno 1 musí načíst aktuální stav (hlavu `A` s razítkem `13` a nový `next`, kterým je teď `C`) a zkusit to znovu.

**Důležité detaily**

* `stampHolder`: Metoda `get` u `AtomicStampedReference` bohužel nemůže vrátit dvě hodnoty najednou (objekt a int), proto se v Javě používá toto jednoprvkové pole, do kterého metoda "přibalí" aktuální verzi razítka.
* Garance: Tato struktura je tzv. Lock-Free. To znamená, že i když se jedno vlákno zasekne, nezablokuje ostatní v práci (na rozdíl od `synchronized`).

## Dodatečná témata

### AtomicStampedReference a get() jako pole a ne int/Integer?

V Javě nemáme možnost vrátit z metody dvě různé hodnoty (objekt i primitivní `int`) najednou, aniž bychom vytvářeli nový pomocný objekt (což by v rychlém CAS cyklu zbytečně zatěžovalo Garbage Collector). Vývojáři JDK to tedy vyřešili tímto „trikem“:

* Předá se pole: „krabička“ (pole o jedné položce).
* Metoda ji naplní: `head.get(stampHolder)` se podívá do svých vnitřností, vrátí referenci na `Node` jako výsledek funkce a do pole zapíše aktuální číslo verze.
* Výstupní parametr: Funguje to tedy v podstatě jako `out` parametr v C# nebo předání ukazatele v C++.

Kdyby Java v té době měla Records nebo lepší Value Types, možná by to vypadalo jinak. Takhle je to ale nejefektivnější cesta z pohledu výkonu:

* Žádná alokace v cyklu: Pole `stampHolder` se vytvoří jednou před cyklem `while`.
* Rychlost: Zápis do pole je extrémně rychlá operace.
* Atomičnost: Klíčové je, že `AtomicStampedReference` uvnitř zajistí, aby náhled na objekt i na razítko proběhl v jeden logický moment, takže se nestane, že by se vrátil objekt z verze 10 a razítko z verze 11.

#### Proč nepoužít `Thread.sleep()`?

I když se může zdát, že `LockSupport.parkNanos()` a `Thread.sleep()` řeší podobný problém, jejich účel i chování se zásadně liší.

**Sémantika**

* `Thread.sleep()` vyjadřuje:\
  „Vlákno má být uspáno přesně (minimáně) na dobu `X` milisekund.“
* `LockSupport.park()` vyjadřuje:\
  „Vlákno má být pozastaveno, dokud nedojde k jeho odblokování (`unpark`) nebo nevyprší časový limit.“

**Reaktivita**

Příkaz `LockSupport.park()` je navržen pro nízkoúrovňové synchronizační mechanismy. Vlákno může být velmi snadno probuzeno z jiného vlákna pomocí `LockSupport.unpark(thread)`. Oproti tomu `Thread.sleep()` vyžaduje přerušení pomocí `thread.interrupt()`. To představuje méně přehledný, náročnější a méně vhodný mechanismus pro řízení synchronizace.

**Přesnost a režie**

`LockSupport.park()` (resp. `parkNanos(...)`) poskytuje efektivnější způsob čekání v rámci JVM, zejména pro tzv. _spin-wait_ scénáře.

{% code lineNumbers="true" %}
```
LockSupport.parkNanos(100);
```
{% endcode %}

Takto krátká doba (např. 100 ns) je u `sleep()` prakticky nedosažitelná. `Thread.sleep()` je omezen rozlišením operačního systému (typicky \~1 ms, v některých případech až \~15 ms).

Kvůli tomu by vlákno bylo uspáno výrazně déle, než je požadováno a došlo by k výraznému snížení propustnosti (_throughput_), například u lock-free zásobníku

### wait() / notify() vs spin-wait

Rozdíl mezi `wait` a `spin-wait` (blokující vs neblokující kód) je v tom, kde a jak vlákno čeká.

**1. `wait()` / `notify()` (Vláknové parkování)**

Když zavoláš `wait()`, říkáš operačnímu systému: _"Teď nemám co dělat, úplně mě vypni a vzbuď mě, až se něco změní."_

* Mechanismus: Vlákno je vyřazeno z plánovače CPU. Přestane spotřebovávat procesorový čas.
* Context Switch: Aby se vlákno uspalo a později vzbudilo, musí operační systém provést tzv. přepnutí kontextu (uložit registry, stav vlákna atd.). To je relativně "drahá" operace (trvá mikrosekundy).
* Kdy použít: Když očekáváš, že čekání bude trvat dlouho (milisekundy a více).

**2. Spin-wait (Točení v kruhu)**

Vlákno v cyklu (např. `while(true)`) neustále dokola kontroluje podmínku.

* Mechanismus: Vlákno stále běží na CPU na 100 %. Procesor "pálí" instrukce jen proto, aby zjistil, že se nic nezměnilo.
* Latence: Je extrémně nízká. Jakmile se data objeví, vlákno reaguje okamžitě (v řádu nanosekund), protože nemusí čekat, až ho OS "nastartuje".
* Kdy použít: Jen tehdy, pokud očekáváš, že čekání bude extrémně krátké (méně než doba přepnutí kontextu).

| **Vlastnost**   | `wait()` (Blocking)              | Spin-wait (Non-blocking)                 |
| --------------- | -------------------------------- | ---------------------------------------- |
| Spotřeba CPU    | Nulová (vlákno spí)              | Maximální (vlákno maká naprázdno)        |
| Rychlost reakce | Pomalejší (režie OS)             | Okamžitá (nanosekundy)                   |
| Vhodné pro      | Dlouhé čekání, vysoká konkurence | Velmi krátké čekání, low-latency systémy |
| Implementace    | `synchronized`, `Lock`           | `while` smyčka + `Atomic` proměnné       |

**Hybridní přístup v Javě**

Protože čistý spin-wait je "vrah" baterií a procesorů, v praxi (i v našem kódu) se používá adaptivní spin:

1. Vlákno chvíli "spinuje" (zkouší to párkrát dokola).
2. Pokud se stále nic neděje, použije se `Thread.onSpinWait()` (instrukce pro CPU, aby trochu zvolnilo) nebo `LockSupport.parkNanos()`.
3. Nakonec se vlákno uspí úplně.

{% hint style="info" %}
Samotné JVM (Java Virtual Machine) interně používá spin-wait, než tě uspí přes `synchronized`. Než tě "vypne", zkusí na pár nanosekund spinovat, jestli náhodou ten zámek jiný procesor zrovna neuvolnil.
{% endhint %}
