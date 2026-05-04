# Simple Demo

Jednoduchá demonstrace rozdílu ve výkonnosti klasický vs. virtuálních vláken.

## Kontext

Budeme demonstrovat, jak dlouho trvá vytvořit množství malinkých vláken, která provedou minimální operaci a zase zaniknou. Cílem bude zjistit časovou náročnost vytvoření a běhu takových vláken.

## Prerekvizity

Budeme mít pomocnou funkci na jednoduché čekání ukončení vlákna:

{% code lineNumbers="true" %}
```java
  private static void joinIgnoringException(Thread t) {
    try {
      t.join();
    } catch (InterruptedException e) {
      // intentionally blank
    }
  }
```
{% endcode %}

Dále budeme mít dvě konstanty ovlivňující chování aplikace - kolik má být vláken a jak dlouho mají zdržovat - obojí v milisekundách:

{% code lineNumbers="true" %}
```java
  private static final int THREAD_COUNT = 10_000;
  private static final int THREAD_SLEEP_TIME = 10;
```
{% endcode %}

Dále uvažujme jednoduchý kód, který by například načítal data z externího zdroje a po chvíli zdržení je vracel zpět. V našem příkladu toto budeme simulovat pouze krátkým (fixním) zdržením a bez vracení hodnoty:

{% code lineNumbers="true" %}
```java
  private static void doWork(){
    try {
      Thread.sleep(THREAD_SLEEP_TIME);
    } catch (InterruptedException e) {
      // intentionally blank
    }
  }
```
{% endcode %}

A konečně jednoduchou `main` funkci, která vyvolá obě budoucí funkcionality:

{% code lineNumbers="true" %}
```java
  public static void main(String[] args) {
    runThreads();
    runVirtualThreads();
  }
```
{% endcode %}

## Common Threading

Nejdříve jednoduchá funkce, která vytvoří klasická vlákna. Protože máme pro běh pouze funkci, bude nám stačit využití klasické třídy `Thread` a předání pracovní funkce jako instance rozhraní `Runnable` při vytváření vláken. Následuje jednoduché spuštění a ukončení. Skrz funkci probíhá také měření a vypisování času běhu.

{% code lineNumbers="true" %}
```java
  private static void runThreads() {
    long start;
    long end;

    System.out.println("Normal threads");

    start = System.nanoTime();
    List<Thread> threads = new ArrayList<>(THREAD_COUNT);

    for (int i = 0; i < THREAD_COUNT; i++) {
      threads.add(new Thread(Program::doWork));
    }
    end = System.nanoTime();
    System.out.printf("Initialized in %d\n", (end - start) / 1_000_000);

    start = System.nanoTime();
    threads.forEach(Thread::start);
    threads.forEach(Program::joinIgnoringException);
    end = System.nanoTime();
    System.out.printf("Finished in %d\n", (end - start) / 1_000_000);

    System.out.println("Done");
    System.out.println();
  }
```
{% endcode %}

## Virtual Threading

Pro virtuální přístup se zásadně změní pouze způsob vytváření vláken. Budeme je vytvářet jako virtuální.

{% hint style="info" %}
Pro vytvoření a spuštění rovnou bychom mohli používat přímo konstrukci `Thread.startVirtualThread(Program::doWork);` nebo `Thread.ofVirtual().start(Program::doWork);` . My však chceme měřit i čas spuštění a ukončení vlákna, proto je nejdříve pouze vytvoříme a spustíme si je ručně až později.
{% endhint %}

Další část kódu už zůstává stejná, vlákna spustíme a počkáme na dokončení:

{% code lineNumbers="true" %}
```java
  private static void runVirtualThreads() {
    long start;
    long end;

    System.out.println("Virtual threads");

    start = System.nanoTime();
    List<Thread> threads = new ArrayList<>(THREAD_COUNT);

    for (int i = 0; i < THREAD_COUNT; i++) {
      threads.add(Thread.ofVirtual().unstarted(Program::doWork)); // <- vytvoření
    }
    end = System.nanoTime();
    System.out.printf("Initialized in %d\n", (end - start) / 1_000_000);

    start = System.nanoTime();
    threads.forEach(Thread::start);
    threads.forEach(Program::joinIgnoringException);
    end = System.nanoTime();
    System.out.printf("Finished in %d\n", (end - start) / 1_000_000);

    System.out.println("Done");
    System.out.println();
  }
```
{% endcode %}

## Kompletní kód

{% code lineNumbers="true" %}
```java
import java.util.ArrayList;
import java.util.List;


public class Program {

  private static final int THREAD_COUNT = 10_000;
  private static final int THREAD_SLEEP_TIME = 10;

  public static void main(String[] args) {
    runThreads();
    runVirtualThreads();
  }

  private static void runVirtualThreads() {
    long start;
    long end;

    System.out.println("Virtual threads");

    start = System.nanoTime();
    List<Thread> threads = new ArrayList<>(THREAD_COUNT);

    for (int i = 0; i < THREAD_COUNT; i++) {
      threads.add(Thread.ofVirtual().unstarted(Program::doWork));
    }
    end = System.nanoTime();
    System.out.printf("Initialized in %d\n", (end - start) / 1_000_000);

    start = System.nanoTime();
    threads.forEach(Thread::start);
    threads.forEach(Program::joinIgnoringException);
    end = System.nanoTime();
    System.out.printf("Finished in %d\n", (end - start) / 1_000_000);

    System.out.println("Done");
    System.out.println();
  }

  private static void runThreads() {
    long start;
    long end;

    System.out.println("Normal threads");

    start = System.nanoTime();
    List<Thread> threads = new ArrayList<>(THREAD_COUNT);

    for (int i = 0; i < THREAD_COUNT; i++) {
      threads.add(new Thread(Program::doWork));
    }
    end = System.nanoTime();
    System.out.printf("Initialized in %d\n", (end - start) / 1_000_000);

    start = System.nanoTime();
    threads.forEach(Thread::start);
    threads.forEach(Program::joinIgnoringException);
    end = System.nanoTime();
    System.out.printf("Finished in %d\n", (end - start) / 1_000_000);

    System.out.println("Done");
    System.out.println();
  }

  private static void joinIgnoringException(Thread t) {
    try {
      t.join();
    } catch (InterruptedException e) {
      // intentionally blank
    }
  }

  private static void doWork(){
    try {
      Thread.sleep(THREAD_SLEEP_TIME);
    } catch (InterruptedException e) {
      // intentionally blank
    }
  }

}
```
{% endcode %}
