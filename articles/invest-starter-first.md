# Если вы хотите написать проект в котором каким-либо образом есть рыночные данные, знакомы с java или kotlin и слышали про Spring Boot то эта статья для вас.

Я являюсь автором [Spring Boot стартера](https://github.com/Dankosik/invest-api-java-sdk-starter) с помощью которого можно легко интегрировать TinkoffInvestApi в свои Spring Boot  приложения. В стартере упор сделан на стриминг данных - пишите только логику обработки, за все остальное позаботится стартер.

В этой статье я хочу лишь познакомить вас с `Invest Api Java Sdk Starter`, показать идеи и часть возможностей. Без деталей реализации, они будут в серии следующих статей.
## Пререквезиты
- Нужно быть клиентом Тинькофф, так как мы будем использовать токен Тинькофф Инвестиций
- `jdk17+`
- `Maven 3+` либо `Gradle 8.5+` (если `jdk21+`) и `Gradle 7.3+` (если `jdk17+`)
- `SpringBoot 3.0+`

Для тех кто не знаком с TinkoffInvestApi - можете ознакомиться на их [страничке](https://github.com/RussianInvestments/investAPI) в GitHub.
Если коротко: Tinkoff Invest API — это интерфейс для взаимодействия с торговой платформой.

### Какие задачи можно решать:
- Анализ котировок бумаг
- Сигналы на покупку или продажу (Например кидать оповещение о сигнале в Telegram)
- Прогнозы движения акций
- Анализ портфеля
- Автоматизация торговли и создание торговых роботов
- Ведение собственной системы статистики
- Тестирование стратегий на истории

## Какую проблему я вижу в использовании Tinkoff Invest API напрямую?
1) Сложность написания + много boilerplate кода
2) Как следствие высокий порог входа

Если я просто хочу кидать notification в telegram когда доллар на бирже приближается к 100 рублям за шутку мне для этого придеться, помимо написания самой логики отправки notification написать кучу всего большого и страшного, что никак не связано с моей задачей. Такой пример приводят сами разработчики по получению рыночных данных

<details>
  <summary>Пример на java</summary>

Взял его [отсюда](https://github.com/RussianInvestments/invest-api-java-sdk/blob/main/example/src/main/java/ru/tinkoff/piapi/example/Example.java#L157)

  ```java
private static void marketdataStreamExample(InvestApi api) {
    var randomFigi = randomFigi(api, 5);

    //Описываем, что делать с приходящими в стриме данными
    StreamProcessor<MarketDataResponse> processor = response -> {
      if (response.hasTradingStatus()) {
        log.info("Новые данные по статусам: {}", response);
      } else if (response.hasPing()) {
        log.info("пинг сообщение");
      } else if (response.hasCandle()) {
        log.info("Новые данные по свечам: {}", response);
      } else if (response.hasOrderbook()) {
        log.info("Новые данные по стакану: {}", response);
      } else if (response.hasTrade()) {
        log.info("Новые данные по сделкам: {}", response);
      } else if (response.hasSubscribeCandlesResponse()) {
        var subscribeResult = response.getSubscribeCandlesResponse().getCandlesSubscriptionsList().stream()
          .collect(Collectors.groupingBy(el -> el.getSubscriptionStatus().equals(SubscriptionStatus.SUBSCRIPTION_STATUS_SUCCESS), Collectors.counting()));
        logSubscribeStatus("свечи", subscribeResult.getOrDefault(true, 0L), subscribeResult.getOrDefault(false, 0L));
      } else if (response.hasSubscribeInfoResponse()) {
        var subscribeResult = response.getSubscribeInfoResponse().getInfoSubscriptionsList().stream()
          .collect(Collectors.groupingBy(el -> el.getSubscriptionStatus().equals(SubscriptionStatus.SUBSCRIPTION_STATUS_SUCCESS), Collectors.counting()));
        logSubscribeStatus("статусы", subscribeResult.getOrDefault(true, 0L), subscribeResult.getOrDefault(false, 0L));
      } else if (response.hasSubscribeOrderBookResponse()) {
        var subscribeResult = response.getSubscribeOrderBookResponse().getOrderBookSubscriptionsList().stream()
          .collect(Collectors.groupingBy(el -> el.getSubscriptionStatus().equals(SubscriptionStatus.SUBSCRIPTION_STATUS_SUCCESS), Collectors.counting()));
        logSubscribeStatus("стакан", subscribeResult.getOrDefault(true, 0L), subscribeResult.getOrDefault(false, 0L));
      } else if (response.hasSubscribeTradesResponse()) {
        var subscribeResult = response.getSubscribeTradesResponse().getTradeSubscriptionsList().stream()
          .collect(Collectors.groupingBy(el -> el.getSubscriptionStatus().equals(SubscriptionStatus.SUBSCRIPTION_STATUS_SUCCESS), Collectors.counting()));
        logSubscribeStatus("сделки", subscribeResult.getOrDefault(true, 0L), subscribeResult.getOrDefault(false, 0L));
      } else if (response.hasSubscribeLastPriceResponse()) {
        var subscribeResult = response.getSubscribeLastPriceResponse().getLastPriceSubscriptionsList().stream()
          .collect(Collectors.groupingBy(el -> el.getSubscriptionStatus().equals(SubscriptionStatus.SUBSCRIPTION_STATUS_SUCCESS), Collectors.counting()));
        logSubscribeStatus("последние цены", subscribeResult.getOrDefault(true, 0L), subscribeResult.getOrDefault(false, 0L));
      }
    };
    Consumer<Throwable> onErrorCallback = error -> log.error(error.toString());

    //Подписка на список инструментов. Не блокирующий вызов
    //При необходимости обработки ошибок (реконнект по вине сервера или клиента), рекомендуется сделать onErrorCallback
    api.getMarketDataStreamService().newStream("trades_stream", processor, onErrorCallback).subscribeTrades(randomFigi);
    api.getMarketDataStreamService().newStream("candles_stream", processor, onErrorCallback).subscribeCandles(randomFigi);
    api.getMarketDataStreamService().newStream("info_stream", processor, onErrorCallback).subscribeInfo(randomFigi);
    api.getMarketDataStreamService().newStream("orderbook_stream", processor, onErrorCallback).subscribeOrderbook(randomFigi);
    api.getMarketDataStreamService().newStream("last_prices_stream", processor, onErrorCallback).subscribeLastPrices(randomFigi);


    //Для стримов стаканов и свечей есть перегруженные методы с дефолтными значениями
    //глубина стакана = 10, интервал свечи = 1 минута
    api.getMarketDataStreamService().getStreamById("trades_stream").subscribeOrderbook(randomFigi);
    api.getMarketDataStreamService().getStreamById("candles_stream").subscribeCandles(randomFigi);
    api.getMarketDataStreamService().getStreamById("candles_stream").cancel();
    //отписываемся от стримов с задержкой
    CompletableFuture.runAsync(()->{

      //Отписка на список инструментов. Не блокирующий вызов
      api.getMarketDataStreamService().getStreamById("trades_stream").unsubscribeTrades(randomFigi);
      api.getMarketDataStreamService().getStreamById("candles_stream").unsubscribeCandles(randomFigi);
      api.getMarketDataStreamService().getStreamById("info_stream").unsubscribeInfo(randomFigi);
      api.getMarketDataStreamService().getStreamById("orderbook_stream").unsubscribeOrderbook(randomFigi);
      api.getMarketDataStreamService().getStreamById("last_prices_stream").unsubscribeLastPrices(randomFigi);

      //закрытие стрима
      api.getMarketDataStreamService().getStreamById("candles_stream").cancel();

    }, delayedExecutor)
      .thenRun(()->log.info("market data unsubscribe done"));


    //Каждый marketdata стрим может отдавать информацию максимум по 300 инструментам
    //Если нужно подписаться на большее количество, есть 2 варианта:
    // - открыть новый стрим
    api.getMarketDataStreamService().newStream("new_stream", processor, onErrorCallback).subscribeCandles(randomFigi);
    // - отписаться от инструментов в существующем стриме, освободив место под новые
    api.getMarketDataStreamService().getStreamById("new_stream").unsubscribeCandles(randomFigi);

    //При вызове newStream с id уже подписаного приведет к пересозданию стрима с версии 1.4
    api.getMarketDataStreamService().newStream("candles_stream", processor, onErrorCallback)
      .subscribeCandles(randomFigi);
  }
```
</details>

Выглядит непонятным и как новичку во всем этом разобраться. Я предлагаю использовать мой стартер.

Вот так будет выглядеть получения последних цен доллара и их обработка:
```java
@HandleLastPrice(ticker = "USDRUB")
class DollarLastPriceHandler implements BlockingLastPriceHandler {

    @Override
    public void handleBlocking(LastPrice lastPrice) {
       //отправляем notification в telegram когда цена == 100
    }
}
```

Вся внутренняя реализаци скрыта, просто пишите что вы будете делать каждый раз когда изменяется цена, `handleBlocking` будет вызываться каждый раз когда изменяется цена. Кстати если вы используете jdk21+ BlockingLastPriceHandler будет исполнен на виртуальном потоке. Если jdk ниже 21 версии рекомендую использовать

```java
@HandleLastPrice(ticker = "USDRUB")
class DollarLastPriceAsyncHandler implements AsyncLastPriceHandler {

    @Override
    public CompletableFuture<Void> handleAsync(LastPrice lastPrice) {
        return CompletableFuture.runAsync(() ->  //отправляем notification в telegram когда цена == 100);
    }
}
```

Все хендлеры будут созданы как компоненты spring, поэтому можно отдельно написать условный TelegramService и инжектить его в любой хендлер
```java
@HandleLastPrice(ticker = "USDRUB")
class DollarLastPriceHandler implements BlockingLastPriceHandler {
    private TelegramService telegramService;

    public void DollarLastPriceHandler(TelegramService telegramService) {
       this.telegramService = telegramService;
    }

    @Override
    public void handleBlocking(LastPrice lastPrice) {
       //отправляем notification в telegram когда цена == 100
       //telegramService.sendNotification()
    }
}
```

Обрабатывать сделки, стаканы, последние цены, обновление портфеля и т.д. можно аналогично. Отличие будет в названии аннотаций и интерфейсов. Подробнее можно ознакомиться в ридми и документации. Также есть два демо проекта:

[На kotlin + gradle.kts](
https://github.com/Dankosik/invest-starter-demo/blob/main/src/main/kotlin/io/github/dankosik/investstarterdemo/InvestStarterDemoApplication.kt#L65)
[На java + maven](
https://github.com/Dankosik/invest-starter-demo-java/blob/main/src/main/java/io/github/dankosik/investstarterdemojava/InvestStarterDemoJavaApplication.java#L44)


## Чтобы это все заработало:
1) Я бы рекомендовал почитать про возможности, понятия и определения в официальной [документации](https://russianinvestments.github.io/investAPI/) Tinkoff Invest API. Если вы уже с ней знакомы можно пропустить этот пункт
2) Создаем любым удобным способом spring boot проект и подключаем зависимости:
<details>
  <summary>Для build.gradle.kts</summary>

```groovy
implementation("io.github.dankosik:invest-api-java-sdk-starter:0.6.1-beta40")
```
И понадобится одна из:
```groovy
implementation("org.springframework.boot:spring-boot-starter-web")
```
```groovy
implementation("org.springframework.boot:spring-boot-starter-webflux")
```

</details>

<details>
  <summary>Для build.gradle</summary>

```groovy
implementation 'io.github.dankosik:invest-api-java-sdk-starter:0.6.1-beta40'
```
И понадобится одна из:
```groovy
implementation 'org.springframework.boot:spring-boot-starter-web'
```
```groovy
implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

</details>

<details>
  <summary>Для Maven</summary>

```xml
<dependency>
    <groupId>io.github.dankosik</groupId>
    <artifactId>invest-api-java-sdk-starter</artifactId>
    <version>0.6.1-beta40</version>
    <classifier>plain</classifier>
</dependency>
```
И понадобится одна из:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```
</details>

3) Всталяем это в `application.yml` заменяя токен на реальный ([как получить токен](https://russianinvestments.github.io/investAPI/token/))
```yaml
tinkoff:
  starter:
    apiToken:
      fullAccess:
        "ваш токен"
```
4) Берем любой понравившийся пример из демо проектов или как вариант печатаем в консоль все сделки по СБЕРУ
```java
@HandleTrade(ticker = "SBER")
class BlockingSberTradeHandler implements BlockingTradeHandler {

    @Override
    public void handleBlocking(@NotNull Trade trade) {
        System.out.println(trade);
    }
}
```
## Дальнейшие планы:
- Написание доки/wiki по стартеру.
- Поддержка Tinkoff Invest API версий 1.7 и 1.8 (так как сейчас стартер работает на версии 1.6)
- После получения фидбека буду пилить новые фичи
# Полезные ссылки:
- [Документация](https://russianinvestments.github.io/investAPI/) Tinkoff Invest API
- [Ссылка](https://github.com/Dankosik/invest-api-java-sdk-starter) на стартер