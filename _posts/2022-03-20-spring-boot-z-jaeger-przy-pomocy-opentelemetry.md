---
layout: post
title:  "Spring Boot z Jaeger przy pomocy OpenTelemetry"
date:   2022-03-19 15:00:00 +0100
categories: opentelemetry
---

Do stworzenie projektu wykorzystam Spring Boota i w pierwszej kolejności narzędzie Jaeger. Najpierw jednak wyjasnie podstawowe koncepcje, które posłużą jako baza do dalszych rozważań. 

Span - zawiera się w Trace i reprezentuje pojedyncza operacje w systemie rozproszonym. Zawiera nazwe, czas początku i końca, SpanContext i zestaw dodatkowych atrybutów. 

Trace - agreguje wiele Spans, które tworzą  w całości biznesową operacje. Zaczyna się od głównego spana a następnie ten moze posiadać pod sobą kolejne reprezentujace kolejne pojedyńcze operacje. 

Wsród narzędzi, które można było wykorzystać do zbierania informacji o komunikacji pomiedzy serwisami jest Zipkin i Jaeger. Architektura obu rozwiązań jest bardzo podobna. Ja zdecydowałem się na wykorzystanie Jaeger, ponieważ chciałem poznać to narzędzie. 

Na samym początku potrzebny jest serwer Jaeger. Ja wykorzystam obraz dockera all-in-one, perfekcyjny dla debugowania i developmentu. Można uruchomić go komendą:

```bash
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.26

```

Aby upewnić się, że serwer jest już gotowy, można spróbować podłączyć się pod UI, który jest dostępny pod adresem [http://localhost:16686](http://localhost:16686/).  Gdy wszystko jest w porzadku, można przejsć do następnego kroku którym jest stworzenie kilku aplikacji Spring Boot komunikujących się ze sobą. Ja żeby sobie uprościć zadanie, zdecydowałem się na stworzenie jednej aplikacji, która będzie miała możliwość przekierowania pod następny adres ustawiony w application.properties. Cała implementacja znajduje się w ForwardController.

```java
@RestController
public class ForwardController {

    @Value("${next.service:}")
    private String nextService;

    private RestTemplate restTemplate = new RestTemplate();

    @GetMapping("/forward")
    public void forward(HttpServletResponse response) throws IOException {
        if (!StringUtils.isEmpty(nextService)) {
            restTemplate.getForEntity(nextService, Void.class);
        }
    }

    @GetMapping("/slow-forward/{sleep}")
    public void slowForward(@PathVariable long sleep, HttpServletResponse response) throws IOException, InterruptedException {
        TimeUnit.SECONDS.sleep(sleep);
        if (!StringUtils.isEmpty(nextService)) {
            restTemplate.getForEntity(nextService, Void.class);
        }
    }

    @GetMapping("/stop")
    public String stop() {
        return "Thanks, bye";
    }

    @GetMapping("/stop-error")
    public void stopError(HttpServletResponse response) {
        response.setStatus(500);
    }
}
```

Z powyższego kodu wynika, że każdy request wykonany na adres /forward wyzwoli kolejne zapytanie pod nextService. Mając tak skonstruowany kontroler, moge stworzyć łańcu powiązań pomiedzy usługami symulując różne sytujace. Napisany serwis uruchomie cztery razy i bede symulować ruch z pomocą powyższego kontrolera.

```bash
-javaagent:opentelemetry-javaagent-all.jar
-Dotel.resource.attributes=service.name=service-b
-Dotel.traces.exporter=jaeger
```

Jako, że w przypadku tego demo, nie bede eksportować metryk do żadnego systemu, ustawiam flage otel.metrics.exporter na none, dzieki czemu podczas działania agent nie będzie informować o błędzie:

> [otel.javaagent 2021-09-12 20:57:12:157 +0200] [grpc-default-executor-2] ERROR io.opentelemetry.exporter.otlp.metrics.OtlpGrpcMetricExporter - Failed to export metrics. Server is UNAVAILABLE. Make sure your collector is running and reachable from this network.UNAVAILABLE: io exception
> 

Co wiecej, korzystam z domyślnej konfiguracji agenta, który łączy się z Jaegerem pod adresem http://localhost:14250. W przypadku gdyby jednak te dane były inne, to można skorzystać z tego propertisa

> otel.exporter.jaeger.endpoint=http://host:port
> 

Przykładowa sytuacja, ktorą można zasymulować prezentuje się następująco: Są cztery serwisy i rozmawiają miedzy sobą. Cały łanchu zaczyna się od requestu na [http://localhost:8080/slow-forward/3](http://localhost:8080/slow-forward/3) a następnie są wysyłane kolejne zapytania jak w tabeli poniżej.  



| Serwis      | Zapytanie przychodzące | Następny serwis |
| ----------- | ---------------------- | --------------- |
| Serwis A    | http://localhost:8080/slow-forward/3       | http://localhost:8082/forward
| Serwis B    | http://localhost:8082/forward        | http://localhost:8083/slow-forward/10
| Serwis C    | http://localhost:8083/slow-forward/10        | http://localhost:8084/stop
| Serwis D    | http://localhost:8084/stop        | Brak nastepnego zapytania - koniec


Taki łancuch wydarzeń wygeneruje następny trace. 

![Widok wszystkich zapytań](/images/OpenTelemetry/2/traces1.png)

![Poszczególne zapytanie](/images/OpenTelemetry/2/traces2.png)

Dzieki narzędziu można zobaczyć ile trwała łącznie cała transakcja biznesowo, ile zajął poszczególny request, przez jaki kontroler przeszedł i jakim wynikiem zakończyła się cała operacja. Według mnie,  wykorzystanie tego typu narzędzia pozwala dobrze zwizualizować przez jakie serwisy przechodząc requesty i czy gdzieś nie ma problemów ze obsługą zapytań. 