---
layout: post
title:  "Wprowadzenie do open telemetry"
date:   2022-01-09 15:00:00 +0100
categories: opentelemetry
---

Jednym z możliwych podejść przy tworzeniu dzisiejszych architektur systemów informatycznych jest podziałał funkcjonalności na mikrousługi. Takie podejście oprócz zalet niesie ze sobą także wyzwania. Jednym z koniecznych podejść do zastosowania w takiej architekturze jest obserwowalność czyli raportowanie wewnętrznego stanu aplikacji na zewnątrz. Można to osiągnąć za pomocą takich wzorców jak:

- Zbieranie logów (Log aggregation) - każda usługa generuje logi, a następnie te logi są wysyłane do scentralizowanego systemu, który upraszcza ich przeglądanie w poszukiwaniu problemów. Przykłady narzędzi: Graylog, Logstash

- Metryki - aplikacja udostępnia informacje o swoim stanie zarówno technicznym jak i biznesowym. Może wystąpić w dwóch podejściach.
    - Push - Mikroserwis wysyła metryki do usługi agregującej metryki
    - Pull - Usługa agregująca metryki pobieranie metryki z mikroserwisu.  

    **Przykłady narzędzi: Prometheus, Grafana, SigNoz**

- Rozproszone sledzenie (Distributed tracing) - śledzenie ruchu pomiedzy usługami dzięki czemu czemu można zorientować się czy komunikacja pomiedzy usługami nie trwa za długo. Implementacja tego podejścia odbywa się poprzez przydzielenie unikalnego indetyfikatora zapytania (trace id), a nastepnie przesyłanie go pomiedzy serwisami. Samo zbieranie ruchu najcześciej nie odpowie nia pytanie dlaczego aplikacja spowolniła, od tego służą metryki i logi.
**Przykłady narzędzi: Jaeger, Zipkin**

- Health check - podejście w którym mikroserwis raportuje swój stan. W momencie, kiedy wystąpi jakikolwiek problem, aplikacja przechodzi w stan failed i requesty nie są kierowane w strone mikroserwisu, który ma problem.   
    **Przykłady narzędzi: Actuator**

Do zaimplementowania tych wzorców jest przynajmniej kilka różnych narzędzi i wymagają one innej konfiguracji, w szczególności po stronie klienta (mikroserwisu) wysyłającego informacje do serwera. Mnogość różnych rozwiązań może powodować problemy z utrzymywaniem konfiguracji dlatego przy tym problemie można rozwazyć OpenTelemetry. Jest to zbiór narzędzi, API i SDK służących do zbierania danych telemetrycznych takich jak metryki, logi i traces. Projekt powstał z powodu braku standaryzacji w tym obszarze, wiele różnych rozwiązań powodowało problemy z utrzymywaniem konfiguracji dla poszczególnych usług. Co ważne, OpenTelemtry nie ogranicza się tylko do samej Javy a także inne jezyki są przez to wspierane. Może to być istotne dla organizacji, które robią mikroserwisy w różnych językach programowania i ekosystemach.

## Status Specyfikacji Open Telemetry

Open Telemetry wspiera następujące wzroce obserwowalności. 

| Wzorzec      | Status                     |
| -----------  | -----------                |
| Tracing      | Stabilne, long term support|
| Metrics      | Cały czas rozwijane        |
| Logging      | Wczesny etap               |


W powyżej tabeli widać, że każdy z wzorców znajduje się na innym etapie rozwoju i tak naprawde jedynie Tracing jest w statusie stabilinym. Wiecej informacji dostępnych jest pod tym linkiem: [https://opentelemetry.io/status/](https://opentelemetry.io/status/)

## Zbieranie danych z OpenTelemtry

W przypadku Javy, OpenTelemtry wykorzystuje "OpenTelemetry Java Agent" do zbierania informacji. Najprostrzy sposób aby połączyć java agenta ze swoją aplikacja jest nastepujacy:

 

```bash
java -javaagent:path/to/opentelemetry-javaagent-all.jar \
     -jar myapp.jar
```

Domyślnie, OpenTelemetry Java Agent wykorzystuje OpenTelemetry Protocol (OTLP) do przesyłania danych telemetrycznych jednak można bez problemu przekonfigurować tego agenta na konkretne rozwiązania. Na moment pisania artykułu, następujące eksportery są wspierane:

- OLTP
- Jaeger
- Zipkin
- Prometheus
- Logging

Parametry konfiguracyjne sa przekazywane z pomocą flagi -D. 

```bash
java -javaagent:path/to/opentelemetry-javaagent-all.jar \
     -Dotel.resource.attributes=service.name=your-service-name \
     -Dotel.traces.exporter=zipkin \
     -jar myapp.jar
```

Parametry do konfiguracji OLTP i innych eksporerów są dostępne pod tym linkiem: [https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/agent-config.md](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/agent-config.md)

Wyzwania związane z obserwowalnością i próby ich rozwiązania zyskują na coraz wiekszym zainteresowaniu. Jedną z nowości wersji Spring 6 będzie projekt Spring Observability, który także ustandaryzuje podejście do tego tematu w ekosystemie Spring Boota. (Autorzy będą czerpać z doświadczeń pochodzących z projektu Spring Cloud Sleuth) Na ten moment jednak skorzystam z narzędzia OpenTelemetry i w następnym artykule stworze kilka serwisów, które beda wymieniać się danymi i na tej podstawie spróbuje zaimplementować do nich wzorzec obserwowalnosci. 

## Źródła

[https://medium.com/@greekykhs/microservices-observability-patterns-eff92365e2a8](https://medium.com/@greekykhs/microservices-observability-patterns-eff92365e2a8)