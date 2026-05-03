# Demo

TODO

```
import lombok.Getter;
import lombok.Setter;

import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Main {
  private final static Model model = new Model();
  private final static ExecutorService updateExecutor = Executors.newSingleThreadExecutor();

  public static void main(String[] args) {
    updateModelView();
    DataProvider.getModelInfoAsync()
            .thenApply(result -> {
              model.setTitle(result.title());
              model.setIcao(result.icao());
              return result;
            })
            .thenCompose(result ->
                    CompletableFuture
                            .runAsync(Main::updateModelView, updateExecutor)
                            .thenApply(v -> result)
            )
            .thenCompose(result -> DataProvider.getCompanyAirplanesAsync(result.icao()))
            .thenAccept(result -> {
              List<Airplane> tmp = result.stream().map(q -> {
                        Airplane ret = new Airplane();
                        ret.setRegistration(q);
                        return ret;
                      })
                      .toList();
              model.setAirplanes(tmp);
            })
            .thenRunAsync(() -> CompletableFuture.runAsync(Main::updateModelView, updateExecutor))
            .thenCompose(v -> {
              List<CompletableFuture<Void>> lst = new ArrayList<>();
              for (Airplane airplane : model.getAirplanes()) {
                CompletableFuture<Void> future = DataProvider.getAirplaneLocationAsync(airplane.getRegistration())
                        .thenAccept(airplane::setLocation)
                        .thenRunAsync(() -> CompletableFuture.runAsync(Main::updateModelView, updateExecutor));
                lst.add(future);
              }

              CompletableFuture<Void>[] arr = lst.toArray(CompletableFuture[]::new);

              CompletableFuture<Void> ret = CompletableFuture.allOf(arr);
              return ret;
            })
            .join();
    updateExecutor.shutdown();
    try {
      updateExecutor.awaitTermination(1, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
      // intentionally blank
    }
    System.out.println("Completed");
  }

  private static void updateModelView() {
    clearConsole();

    System.out.println(" * * *");
    System.out.printf("Airline: %s\n", model.getTitle());
    System.out.printf("\tplanes: %d\n", model.getAirplanes().size());
    for (Airplane airplane : model.getAirplanes()) {
      System.out.printf("\t%s at %s\n", airplane.getRegistration(), airplane.getLocation());
    }
  }

  private static void clearConsole() {
    for (int i = 0; i < 50; i++) {
      System.out.println();
    }
  }
}

class DataProvider {
  private final static Random random = new Random();

  private static int getRandomDelay(int minimumMs, int maximumMs) {
    int ret = random.nextInt(maximumMs - minimumMs + 1) + minimumMs;
    return ret;
  }

  private static void randomSleep(int minimumMs, int maximumMs) {
    int delay = getRandomDelay(minimumMs, maximumMs);
    try {
      Thread.sleep(delay);
    } catch (InterruptedException e) {
      // intentionally blank
    }
  }

  public static AirplaneIcao getModelInfo() {
    randomSleep(500, 1500);
    return new AirplaneIcao("Spirit", "SPR");
  }

  public static CompletableFuture<AirplaneIcao> getModelInfoAsync() {
    CompletableFuture<AirplaneIcao> ret = CompletableFuture.supplyAsync(DataProvider::getModelInfo);
    return ret;
  }

  public static List<String> getCompanyAirplanes(String icao) {
    randomSleep(500, 1500);
    List<String> ret = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
      ret.add(String.format("%s-%d", icao, i + 1));
    }
    return ret;
  }

  public static CompletableFuture<List<String>> getCompanyAirplanesAsync(String icao) {
    CompletableFuture<List<String>> ret = CompletableFuture.supplyAsync(() -> getCompanyAirplanes(icao));
    return ret;
  }

  public static String getAirplaneLocation(String registration) {
    randomSleep(500, 5500);
    return String.format("%s-%d", registration, random.nextInt(100));
  }

  public static CompletableFuture<String> getAirplaneLocationAsync(String registration) {
    CompletableFuture<String> ret = CompletableFuture.supplyAsync(() -> getAirplaneLocation(registration));
    return ret;
  }
}

@Getter
@Setter
class Model {
  private String title = "";
  private String icao = "";
  private List<Airplane> Airplanes = new ArrayList<>();
}

@Getter
@Setter
class Airplane {
  private String registration = "";
  private String location = "";
}

record AirplaneIcao(String title, String icao) {
};
```
