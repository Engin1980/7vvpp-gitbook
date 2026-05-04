# Příklad

{% embed url="https://github.com/Engin1980/7vvpp-futures-example" %}

## Motivace

TODO

## Prerekvizity

Projekt používá anotace `Lombok`.

V projektu se použije jednoduchá funkce vypisující info o aktuálním stavu modelu:

{% code lineNumbers="true" %}
```java
  private static void printCurrent() {
    for (int i = 0; i < 20; i++) {
      System.out.println();
    }

    if (servicesOverview == null) {
      System.out.println("No data yet...");
    } else {
      System.out.println("Source: " + servicesOverview.getSource());
      System.out.println("State: " + servicesOverview.getState());
      if (servicesOverview.getServices() == null)
        System.out.println("\tNo services yet");
      else {
        for (ServiceInfo serviceInfo : servicesOverview.getServices()) {
          System.out.printf("\tService: %s -- %s\n", serviceInfo.getTitle(), serviceInfo.getState());
        }
      }
    }
  }
```
{% endcode %}

Jednoduchá třída main bude obsahovat definici modelu a dvě metody ukazující sekvenční a paralelní získání hodnot:

{% code lineNumbers="true" %}
```java
public class Program {

  private static ServicesOverview servicesOverview;

  public static void main(String[] args) {

    printCurrent();

    runInSequence();

    System.out.println("Done");
  }
  
  private static void printCurrent() { ... }
}
```
{% endcode %}

## Kontext

Uvažujme jednoduchou sadu modelových tříd:

* **ServiceOverview** - který obsahuje informace o zdroji a jednotlivých službách.
* **ServiceInfo** - která obsahuje informace o službě - název a její stav.

{% code lineNumbers="true" %}
```java
@Getter
@Setter
public class ServicesOverview {
  private String source;
  private String state;
  private List<ServiceInfo> services;
}
```
{% endcode %}

{% code lineNumbers="true" %}
```java
@Getter
@Setter
public class ServiceInfo {
  private String title;
  private String state;
}
```
{% endcode %}

Dále budeme mít jednoduchou třídu `DataProvider`, která vrací data z nějakého vzdáleného zdroje - v našem případě budeme zdržení při načtení dat simulovat jednouchým čekáním ve vlákně po náhodně definovanou dobu.

{% code lineNumbers="true" %}
```java
public class DataProvider {

  public ServiceOverviewStateResponse getServiceOverviewState() {
    this.sleepFor(2000, 3500);
    ServiceOverviewStateResponse ret = new ServiceOverviewStateResponse(
            "Ready",
            "Source " + random.nextInt(1, 100)
    );
    return ret;
  }

  public Set<String> getServiceNames(String source) {
    this.sleepFor(2000, 3500);
    Set<String> ret = new HashSet<>();
    for (int i = 0; i < 7; i++) {
      ret.add("Service " + random.nextInt(1, 100));
    }
    return ret;
  }

  public String getServiceState(String title) {
    this.sleepFor(1000, 5000);
    String ret = "Active";
    return ret;
  }

  public record ServiceOverviewStateResponse(String state, String source) {
  }
}
```
{% endcode %}

Protože funkce `getServiceOverviewState()` vrací více hodnot, zabalíme je do návratového typu `ServiceOverviewStateResponse`.

Kompletní výpis výchozího stavu implementace je na: [https://github.com/Engin1980/7vvpp-futures-example/blob/9e13e875e7ca561183ba2e5f0580329652bf29e3/src/eng/futures/DataProvider.java](https://github.com/Engin1980/7vvpp-futures-example/blob/9e13e875e7ca561183ba2e5f0580329652bf29e3/src/eng/futures/DataProvider.java).

## Sekvenční získání hodnot

Nejdříve demonstrujeme základní úlohu - budeme vlastně ukazovat kód nám vytvářené funkce `runInSequcence()`.&#x20;

{% hint style="info" %}
Aby se nám nepletly proměnné, budeme používat bloky pro omezení viditelnosti. V reálném příkladu, pokud bychom chtěli jít při programování tímto směrem, bychom samozřejmě využili funkce.
{% endhint %}

Nejdříve ukázka získání a vypsání obecných informací.

{% code lineNumbers="true" %}
```java
    DataProvider dataProvider = new DataProvider();

    {
      var res = dataProvider.getServiceOverviewState();
      servicesOverview = new ServicesOverview();
      servicesOverview.setSource(res.source());
      servicesOverview.setState(res.state());
      printCurrent();
    }
```
{% endcode %}

Následně ukázka získání názvů všech služeb:

{% code lineNumbers="true" %}
```java
    ...
    {
      var res = dataProvider.getServiceNames(servicesOverview.getSource());
      var lst = res.stream().map(s -> {
        ServiceInfo serviceInfo = new ServiceInfo();
        serviceInfo.setTitle(s);
        serviceInfo.setState("?");
        return serviceInfo;
      }).toList();
      servicesOverview.setServices(lst);
      printCurrent();
    }
```
{% endcode %}

A finálně ukázka získání stavů jednotlivých služeb:

{% code lineNumbers="true" %}
```java
    ...
    {
      for (ServiceInfo serviceInfo : servicesOverview.getServices()) {
        String res = dataProvider.getServiceState(serviceInfo.getTitle());
        serviceInfo.setState(res);
        printCurrent();
      }
    }
```
{% endcode %}

Jak je vidět, toto získání dat je sekvenční - nejdříve se zjistí jedna informace, pak další atd. Zjišťování stavů jednotlivých služeb také probíhá sekvenčně, přestože by stačilo jednoduše zavolat zjištění stavů všech služeb najednou a průběžně (nebo najednou) sesbírat výsledky a aktualizovat model.

Kompletní výpis projektu se sekvenčním řešením je dostupný na: [https://github.com/Engin1980/7vvpp-futures-example/tree/9e13e875e7ca561183ba2e5f0580329652bf29e3](https://github.com/Engin1980/7vvpp-futures-example/tree/9e13e875e7ca561183ba2e5f0580329652bf29e3).

## Vícevláknové získání hodnot

### Úprava pro asynchronní získávání dat

Prvním krokem je rozšíření třídy `DataProvider` získávající data. Ke každé původní metodě (viz výše) doplníme metodu, která bude výsledek vracet zabalený v `CompletableFuture<?>` dle návratového typu původní metody:

{% code lineNumbers="true" %}
```java
package eng.futures;

import java.util.HashSet;
import java.util.Random;
import java.util.Set;
import java.util.concurrent.CompletableFuture;

public class DataProvider {
  ....
  public CompletableFuture<ServiceOverviewStateResponse> getServiceOverviewStateAsync() {
    return CompletableFuture.supplyAsync(this::getServiceOverviewState);
  }

  public CompletableFuture<Set<String>> getServiceNamesAsync(String source) {
    return CompletableFuture.supplyAsync(() -> getServiceNames(source));
  }

  public CompletableFuture<String> getServiceStateAsync(String title) {
    return CompletableFuture.supplyAsync(() -> getServiceState(title));
  }
  ...
}

```
{% endcode %}

### Získávání dat přes CompletableFuture

Nyní upravíme získávání dat.

#### Získání info o ServicesOverview

Nejdříve demonstrace jednoduchého získání dat pro `ServiceOverview`. Nejdříve požádáme o dodání výsledku:

{% code lineNumbers="true" %}
```java
DataProvider dataProvider = new DataProvider();

dataProvider.getServiceOverviewStateAsync()
   ...
```
{% endcode %}

V kódu se volá asynchroní získání dat. Po vrácení výsledku se nad výsledek aplikuje - `thenApply`  - funkce, která vytvoří nový modelový objekt a nastaví mu vlastnosti podle získaných dat.

{% code lineNumbers="true" %}
```java
DataProvider dataProvider = new DataProvider();

dataProvider.getServiceOverviewStateAsync()
        .thenApply(res -> {
          servicesOverview = new ServicesOverview();
          servicesOverview.setSource(res.source());
          servicesOverview.setState(res.state());
          printCurrent();
          return res;
        })
```
{% endcode %}

Samotné vyvolání CompletableFuture způsobí její odpálení bez čekání na opravdové dokončení výsledku.&#x20;

{% hint style="info" %}
Pokud bychom chtěli kód rovnou vyzkoušet, musíme na výsledek volání ještě počkat a doplnit celý příkaz koncem `.join();`
{% endhint %}

#### Získání názvů samostatných služeb

Následně na výsledek do fronty - _chain_ - navážeme získání služeb podle zdroje vráceného z předchozího požadavku.&#x20;

Protože řetězíme na základě předchozích dat další `CompletableFuture`, přidáme navázání přes `thenCompose()`; tento dotaz vrátí seznam názvů služeb (`List<String>`):

{% code lineNumbers="true" %}
```java
...
.thenCompose(res -> dataProvider.getServiceNamesAsync(res.source()))
```
{% endcode %}

A opět, zpracujeme získaná data a vytvoříme z nich seznam služeb, který podstrčíme do modelového objektu:

{% code lineNumbers="true" %}
```java
...
.thenAccept(res -> {
         var lst = res.stream().map(s -> {
           ServiceInfo serviceInfo = new ServiceInfo();
           serviceInfo.setTitle(s);
           serviceInfo.setState("?");
           return serviceInfo;
         }).toList();
         servicesOverview.setServices(lst);
         printCurrent();
       })
```
{% endcode %}

#### Ziskání stavů jednotlivých služeb

Zde bude situace složitější. Obecně potřebujeme projít všechny služby a pro každou zažádat o její stav a výsledek uložit do modelu, což není nijak složité:

{% code lineNumbers="true" %}
```java
.thenRun(_ -> {
    for (ServiceInfo serviceInfo : servicesOverview.getServices()) {
      dataProvider.getServiceStateAsync(serviceInfo.getTitle())
          .thenAccept(res -> {
            serviceInfo.setState(res);
            printCurrent();
          });
    })
```
{% endcode %}

Zde je však důležitý efekt. Vyvolání vnitřních `CompletableFuture` bude asynchronní a nebude se na ně čekat. Odpálí se a už se nesledují. To ale nechceme, my je chceme začlenit do našeho řetězce volání, abychom věděli, kdy se vše dokončí.&#x20;

Proto musíme všechny vytvořené `CompletableFuture` sesbírat (např. do listu) a udělat z nich jeden velký `CompletableFuture`, který bude na ně všechny čekat, a na ten zase budeme čekat my. Protože vracíme `CompletableFuture`, spravující funkce se změní na `thenCompose()`. Tato funkce očekává, že do ní vstupují data, my však žádná data nemáme. Proto vstup nahradíme za prázdný objekt `_`.

{% code lineNumbers="true" %}
```java
...
.thenCompose(_ -> {
  List<CompletableFuture<Void>> tmp = new ArrayList<>();

  for (ServiceInfo serviceInfo : servicesOverview.getServices()) {
    var cf = dataProvider.getServiceStateAsync(serviceInfo.getTitle())
            .thenAccept(res -> {
              serviceInfo.setState(res);
              printCurrent();
            });
    tmp.add(cf);
  }

  CompletableFuture<Void>[] arr = tmp.toArray(new CompletableFuture[0]);
  CompletableFuture<Void> ret = CompletableFuture.allOf(arr); // <- čekání na všechny
  return ret;
})
```
{% endcode %}

Protože potřebujeme na výsledek počkat, doplníme volání ještě o čekání:

{% code lineNumbers="true" %}
```java
...
.join(); // <- čekání na výsledek
```
{% endcode %}

Kompletní výsledný kód:

{% code lineNumbers="true" %}
```java
DataProvider dataProvider = new DataProvider();

dataProvider.getServiceOverviewStateAsync()  // <- vrať overview
        .thenApply(res -> {   // <-zpracuj ho do objektu
          servicesOverview = new ServicesOverview();
          servicesOverview.setSource(res.source());
          servicesOverview.setState(res.state());
          printCurrent();
          return res;
        }).thenCompose(res -> dataProvider.getServiceNamesAsync(res.source()))   // <- vrať služby
        .thenAccept(res -> {   // <- zpracuj názvy služeb
          var lst = res.stream().map(s -> {
            ServiceInfo serviceInfo = new ServiceInfo();
            serviceInfo.setTitle(s);
            serviceInfo.setState("?");
            return serviceInfo;
          }).toList();
          servicesOverview.setServices(lst);
          printCurrent();
        })
        .thenCompose(_ -> {   // <- dotaž se na stavy služeb
          List<CompletableFuture<Void>> tmp = new ArrayList<>();

          for (ServiceInfo serviceInfo : servicesOverview.getServices()) {
            var cf = dataProvider.getServiceStateAsync(serviceInfo.getTitle()) // <- dotaž se na stav konkrétní služby
                    .thenAccept(res -> {   // <- a výsledek zpracuj
                      serviceInfo.setState(res);
                      printCurrent();
                    });
            tmp.add(cf);
          }

          CompletableFuture<Void>[] arr = tmp.toArray(new CompletableFuture[0]);
          CompletableFuture<Void> ret = CompletableFuture.allOf(arr);
          return ret;
        })
        .join();
```
{% endcode %}

