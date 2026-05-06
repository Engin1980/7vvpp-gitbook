# Kolekce - blokovací fronta

## Motivace

Dalším ze způsobů, jak zajistit sdílení dat mezi dvěma vlákny, jsou synchronizované fronty. Tyto fronty jsou odlišné od vláknově bezpečných kolekcí. Vláknově bezpečná kolekce pouze zajišťuje, že k ní v jednu chvíli může přistupovat více vláken a vkládat/vybírat z ní data. Synchronizované fronty umožňují řídit přístup více vláken, tedy zajistit, aby například druhé vlákno mohlo vybrat data z kolekce až v okamžiku, kdy tam první vlákno nějaká data vloží. Tento mechanismus není příliš složité naprogramovat, ale programátor má navíc k dispozici několik implementací, které mu výše uvedené chování zajistí, a tak se s ním nemusí zabývat.

## Implementace

Základní představenou implementací bude jednoduché rozhraní `BlockingQueue`. Jedná se o základní frontu, do které se z jedné strany vkládají data a z druhé strany se data vybírají. První vložená data se vybírají jako první (jedná se o klasické chování fronty).

### Práce s frontou, základní operace

Na rozdíl od základní fronty však existuje několik způsobů, jak lze hodnotu do fronty vložit, a podobně několik způsobů, jak hodnotu z fronty získat. Fronta také může mít určitou (maximální) kapacitu, a pokud je plná, nelze do ní další prvek vložit. Obecně lze chování vkládajících a vybírajících prvků rozdělit na skupiny, že _pokud operaci vložení/vybrání nelze provést, tak_:

* Se vyvolá výjimka
* Vrátí se speciální hodnota, tj. příznak udávající, zda se operace zdařila či nikoliv
* Vlákno provádějící operaci bude pozastaveno až do chvíle, kdy bude možné danou operaci provést
  * Tato varianta může být ještě rozšířena o maximální čas, jaký může dané vlákno čekat.

Základní nabízené operace jsou:

* Vkládání
* Odebírání
* Kontrola ověřující, zda je další prvek k dispozici.

Názvy metod podporujících dané operace shrnuje následující tabulka.

<table data-header-hidden><thead><tr><th width="165" valign="top"></th><th width="123" valign="top">Text</th><th width="119" valign="top"></th><th width="143" valign="top"></th><th width="153" valign="top"></th></tr></thead><tbody><tr><td valign="top"> </td><td valign="top">Výjimka</td><td valign="top">Příznak True/False</td><td valign="top">Pozastavení výpočtu</td><td valign="top">Omezení času čekání</td></tr><tr><td valign="top">Vkládání</td><td valign="top"><code>add(o)</code></td><td valign="top"><code>offer(o)</code></td><td valign="top"><code>put(o)</code></td><td valign="top"><p><code>offer(</code></p><p><code>o, timeout, timeunit)</code></p></td></tr><tr><td valign="top">Odebírání</td><td valign="top"><code>remove(o)</code></td><td valign="top"><code>poll(o)</code></td><td valign="top"><code>take(o)</code></td><td valign="top"><code>poll(timeout, timeunit)</code></td></tr><tr><td valign="top">Test přítomnosti</td><td valign="top"><code>element(o)</code></td><td valign="top"><code>peek(o)</code></td><td valign="top"> </td><td valign="top"> </td></tr></tbody></table>

Programátor si tedy může vybrat, jaké konkrétní chování při operaci s prvkem vyžaduje.

### Nabízené třídy/implementace

Jak bylo řečeno, `BlockingQueue` je pouze rozhraní a pokud programátor potřebuje instanci, může si vybrat některou z implementací:

* `ArrayBlockingQueue` – je fronta založení vnitřně na pevném poli prvků. Při inicializaci musí programátor povinně zadat maximální počet prvků, pro který bude pole určeno; do fronty nepůjdou další prvky vložit, pokud v ní není místo.
* `DelayQueue` – je typ fronty, kdy daný prvek vložený do fronty lze vybrat až po uplynutí určitého času. Dříve se fronta tváří, jako by v ní prvek umístěn nebyl. Prvky se z třídy tedy vybírají podle času vypršení. Do třídy nelze vložit libovolný prvek, ale instanci třídy, která implementuje rozhraní `Delayed`.
* `LinkedBlockingQueue` – je fronta vnitřně založená na obousměrném listu. Je tedy velikostně neomezená (velikost je omezená velikostí dostupné paměti).
* `PriorityBlockingQueue` – je fronta rozšířená o možnosti priority. Vybírání prvků probíhá podle priorit, priorita prvků je dána implementací rozhraní _Comparable_. U prvků, které mají stejnou prioritu, se negarantuje jejich pořadí.
* `SynchronousQueue` – je zvláštním případem fronty, která může mít pouze jeden prvek. Tato fronta se používá jako tzv. _randez-vous_ point pro vlákna, tedy jako místo, kde se vlákna ve svém výpočtu mají setkat. Pokud je ve frontě prvek, nelze další vložit a vkládající vlákno musí počkat. Pokud ve frontě prvek není, nelze (logicky) žádný prvek vybrat a musí se počkat, až nějaké vlákno nějaký prvek vloží.

Samozřejmostí je využití všech výše uvedených tříd spolu s generickými typy – pokud tedy potřebujeme pracovat s datovým typem `String` a do fronty dávat řetězce, použijeme deklaraci proměnné `BlockingQueue<String>`, pokud potřebujeme pracovat s čísly, použijeme například `BlockingQueue<Integer>` atp.

### Poznámka k obousměrné blokovací frontě

Zvláštním případem je obousměrná blokovací fronta, která umožňuje prvky přidávat a vybírat z obou konců libovolně. Má tedy upravené názvy metod pro práci s jednotlivými prvky:

* `addFirst()` vs. `addLast()`
* `offerFirst()` vs. `offerLast()`
* `putFirst()` vs. `putLast()`
* `removeFirst()` vs. `removeLast()`
* `pollFirst()` vs. `pollLast()`
* `takeFirst()` vs. `takeLast()`
* `getFirst()` vs. `getLast()`
* `peekFirst()` vs. `peekLast()`

Obousměrná blokovací fronta je reprezentována rozhraním `BlockingDeque`,  jako implementace v knihovnách Javy existuje jediná třída `LinkedBlockingQueue`.

## Příklad

Jednoduchým příkladem je úloha _Producent-Konzument_, kdy existuje jeden nebo více producentů dodávajících data a jeden nebo více konzumentů tato data spotřebovávající. Producent nemůže dodávat data, pokud je „mezisklad“ mezi producentem a konzumentem plný. Konzument nemůže odebírat data, pokud na „meziskladu“ žádná data nejsou.

Implementace s využitím představených typů je velmi jednoduchá.

{% code lineNumbers="true" %}
```java
// Producent
public class Producer extends EThread {
  BlockingQueue<String> queue;

  public Producer(BlockingQueue<String> queue) {
    this.queue = queue;
  }
  
  @Override
  public void run(){
    super.logPrint("Producer started");
    while (!this.isInterrupted()){
      
      this.trySleepFor(0, 1000);
      
      try {
        super.logPrint("\tProducer adding value");
        queue.put("Next value at " 
          + new java.util.Date().toString());
        super.logPrint("\t\tProducer added value");
      } catch (InterruptedException ex) {
        super.processInterruptedException(ex);
      }
    }
    
    super.logPrint ("Producer finished");
  }
}
```
{% endcode %}

Producent čeká v cyklu náhodně dlouho dobu (až 1 sec) a následně vloží do fronty hodnotu.

<pre class="language-java" data-line-numbers><code class="lang-java"><strong>// Konzument
</strong><strong>public class Consumer extends EThread{
</strong>  
  private BlockingQueue&#x3C;String> queue;
  
  public Consumer(BlockingQueue&#x3C;String> queue){
    this.queue = queue;
  }
  
  @Override
  public void run(){
    super.logPrint("Consumer started");
    
    while(!this.isInterrupted()){
      try {
        
        super.trySleepFor(0, 2000);
        
        super.logPrint("\t Consumer: waiting for data");
        String data = queue.take();
        super.logPrint("DATA: " + data);
      } catch (InterruptedException ex) {
        super.processInterruptedException(ex);
      }
    }
    
    super.logPrint("Consumer finished");
  }
} 
</code></pre>

Konzument v cyklu čeká náhodně až 2 sekundy a poté se pokusí získat z fronty data. Pokud žádná data nejsou, bude čekat, dokud se nějaká data neobjeví.

{% code lineNumbers="true" %}
```java
// Main
public class World {
  public static void main() throws InterruptedException{
    BlockingQueue<String> q = new ArrayBlockingQueue<>(3); // <- délka fronty = 3
    
    Producer p = new Producer(q);
    Consumer c = new Consumer(q);
    
    p.start();
    c.start();
    
    Thread.sleep(20000);
   
    p.interrupt();
    c.interrupt();
    
    p.tryJoin();
    c.tryJoin();
    
    System.out.println("App end");
  }
} 
```
{% endcode %}

Metoda vytvoří frontu, jednoho konzumenta, jednoho producenta, a celý proces spustí na dobu 20 sekund. Protože interval vkládání dat je mnohem rychlejší než vybírání, dojde také k naplnění fronty maximem 3 prvků. Výsledný výstup bude:

{% code lineNumbers="true" %}
```
13:57,608 - Consumer started
13:57,608 - Producer started
13:58,444 - 	 Consumer: waiting for data
13:58,560 - 	Producer adding value
13:58,563 - 		Producer added value
13:58,563 - DATA: Next value at Thu Feb 06 11:13:58 CET 2014
13:58,603 - 	Producer adding value
13:58,603 - 		Producer added value
13:58,939 - 	Producer adding value
13:58,939 - 		Producer added value
13:59,401 - 	Producer adding value
13:59,401 - 		Producer added value
13:59,852 - 	Producer adding value
14:00,440 - 	 Consumer: waiting for data
14:00,440 - 		Producer added value
14:00,440 - DATA: Next value at Thu Feb 06 11:13:58 CET 2014
14:00,763 - 	Producer adding value
14:02,274 - 	 Consumer: waiting for data
14:02,274 - DATA: Next value at Thu Feb 06 11:13:58 CET 2014
...

```
{% endcode %}

Pokud bychom uvažovali šikovnějšího konzumenta, mohli bychom požadovat, že (a to je poměrně typická úloha):

* pokud nejsou data, konzument se po nějakou dobu uspí, a potom
* pokud jsou data, hromadně vyčte všechna data k dispozici najednou.

Zde lze již využít ostatní funkce nabízené třídou `BlockingQueue`.

{% code lineNumbers="true" %}
```java
public class Consumer extends EThread{
  
  private BlockingQueue<String> queue;
  
  public Consumer(BlockingQueue<String> queue){
    this.queue = queue;
  }
  
  @Override
  public void run(){
    super.logPrint("Consumer started");
    
    while(!this.isInterrupted()){
      try {
        
        super.logPrint("Consumer waiting");
        super.trySleepFor(0, 10000);
        
        super.logPrint("Consumer consuming all data");
        // zkusí, zda jsou data
        while (queue.peek() != null){
          String data = queue.take();
          super.logPrint("DATA: " + data);
        }        
        
      } catch (InterruptedException ex) {
        super.processInterruptedException(ex);
      }
    }
    
    super.logPrint("Consumer finished");
  }
} 

```
{% endcode %}

Upravený kód konzumenta ukazuje, jak před každým výběrem dat zkontroluje, zda jsou data k dispozici. Pokud žádná další data nenalezl, přejde do čekacího režimu.

Upravený kód potom bude vypisovat:

{% code lineNumbers="true" %}
```
22:00,174 - Producer started
22:00,174 - Consumer started
22:00,195 - Consumer waiting
22:00,677 - 	Producer adding value
22:00,681 - 		Producer added value
22:00,821 - 	Producer adding value
22:00,821 - 		Producer added value
22:01,636 - 	Producer adding value
22:01,636 - 		Producer added value
22:01,686 - Consumer consuming all data
22:01,686 - DATA: Next value at Thu Feb 06 11:22:00 CET 2014
22:01,686 - DATA: Next value at Thu Feb 06 11:22:00 CET 2014
22:01,686 - DATA: Next value at Thu Feb 06 11:22:01 CET 2014
22:01,686 - Consumer waiting
22:02,628 - 	Producer adding value
22:02,628 - 		Producer added value
22:02,662 - 	Producer adding value
22:02,662 - 		Producer added value
22:03,391 - 	Producer adding value
22:03,391 - 		Producer added value
22:04,293 - 	Producer adding value
22:05,163 - Consumer consuming all data
22:05,163 - 		Producer added value
22:05,163 - DATA: Next value at Thu Feb 06 11:22:02 CET 2014
22:05,163 - DATA: Next value at Thu Feb 06 11:22:02 CET 2014
22:05,163 - DATA: Next value at Thu Feb 06 11:22:03 CET 2014
22:05,163 - DATA: Next value at Thu Feb 06 11:22:04 CET 2014
22:05,163 - Consumer waiting
22:05,277 - 	Producer adding value
22:05,277 - 		Producer added value
22:05,631 - Consumer consuming all data
22:05,631 - DATA: Next value at Thu Feb 06 11:22:05 CET 2014
22:05,631 - Consumer waiting
22:05,965 - 	Producer adding value
22:05,965 - 		Producer added value
22:06,012 - Consumer consuming all data
22:06,013 - DATA: Next value at Thu Feb 06 11:22:05 CET 2014
22:06,014 - Consumer waiting
22:06,496 - 	Producer adding value
22:06,496 - 		Producer added value
...

```
{% endcode %}

U první sady výpisů si lze povšimnout, že konzument opravdu vypsal 3 záznamy – u dalších výpisu však již vypisuje záznamy po čtyřech. Jak je to možné? Vysvětlení poskytne pasáž od času 22:05,631. V průběhu výpis tří záznamů konzumentem některý z producentů stihne ještě jeden záznam vložit dřív, než se konzument stihne uspat.
