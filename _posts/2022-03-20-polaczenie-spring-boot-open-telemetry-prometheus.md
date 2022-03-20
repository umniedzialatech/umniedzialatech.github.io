---
layout: post
title:  "Połączenie Spring Boot, Open Telemetry i Prometheus"
date:   2022-03-19 16:00:00 +0100
categories: opentelemetry
---

Oprócz połączenia z Jaegerem, Open Telemetry można też wykorzystać do integracji z Prometheusem. Co do zasady jednak, ta integracja będzie przebiegać troche inaczej, ponieważ w przypadku Jaegera dane były wysyłane od agenta do usługi. W przypadku Prometheusa, to on będzie pobierać metryki z wystawionego endpointu.  Plan na ten artykuł wygląda następująco:

1) Najpierw skonfigurujemy naszą aplikacje Spring Boot aby wystawiła endpoint do pobrania danych przez Prometheus

2) Pobierzemy i skonfigurujemy Prometheusa aby pobrać logi z wcześniej wystawionego endpointu, a następnie wykorzystamy Grafane do wizualizacji tych danych.

3) Na samym końcu sprawdzimy alternatywne sposoby integracji Spring Boot z Jaegerem i Prometheusem.

 Dla przypomnienia, poprzednio do integracji z Jaegerem wykorzystałem następującą komende: 

```bash
-javaagent:opentelemetry-javaagent-all.jar
-Dotel.resource.attributes=service.name=service-b
-Dotel.traces.exporter=jaeger
```

Teraz zmodyfikuje ją i będzie ona wspierać Prometheusa. Nowa komenda wygląda następująco

```bash
-javaagent:opentelemetry-javaagent-all.jar
-Dotel.resource.attributes=service.name=service-b
-Dotel.traces.exporter=jaeger
**-Dotel.metrics.exporter=prometheus
-Dotel.exporter.prometheus.host=0.0.0.0
-Dotel.exporter.prometheus.port=9464
-Dotel.instrumentation.runtime-metrics.enabled=true**
```

Zaznaczone części są nowe. Teraz jak już mamy nową komende, uruchamiam aplikacje i pod adresem [http://localhost:9464](http://localhost:9464) są dostepne metryki Prometheusa, które prezentują się następująco:

```bash
# HELP runtime_jvm_memory_pool Bytes of a given JVM memory pool.
# TYPE runtime_jvm_memory_pool gauge
runtime_jvm_memory_pool{pool="G1 Survivor Space",type="used",} 1.048576E7
runtime_jvm_memory_pool{pool="CodeHeap 'profiled nmethods'",type="max",} 1.22912768E8
runtime_jvm_memory_pool{pool="G1 Old Gen",type="committed",} 2.43269632E8
runtime_jvm_memory_pool{pool="G1 Eden Space",type="committed",} 2.68435456E8
runtime_jvm_memory_pool{pool="Compressed Class Space",type="max",} 1.073741824E9
runtime_jvm_memory_pool{pool="CodeHeap 'non-nmethods'",type="used",} 1309056.0
runtime_jvm_memory_pool{pool="G1 Survivor Space",type="committed",} 1.048576E7
runtime_jvm_memory_pool{pool="CodeHeap 'non-profiled nmethods'",type="committed",} 3211264.0
runtime_jvm_memory_pool{pool="Compressed Class Space",type="committed",} 6549504.0
runtime_jvm_memory_pool{pool="G1 Old Gen",type="max",} 8.34666496E9
runtime_jvm_memory_pool{pool="G1 Old Gen",type="used",} 7669176.0
runtime_jvm_memory_pool{pool="CodeHeap 'profiled nmethods'",type="committed",} 1.7825792E7
runtime_jvm_memory_pool{pool="CodeHeap 'non-profiled nmethods'",type="max",} 1.22912768E8
runtime_jvm_memory_pool{pool="CodeHeap 'non-nmethods'",type="committed",} 2555904.0
runtime_jvm_memory_pool{pool="Compressed Class Space",type="used",} 5757776.0
runtime_jvm_memory_pool{pool="CodeHeap 'non-nmethods'",type="max",} 5832704.0
runtime_jvm_memory_pool{pool="Metaspace",type="committed",} 4.7443968E7
runtime_jvm_memory_pool{pool="CodeHeap 'profiled nmethods'",type="used",} 1.7771648E7
runtime_jvm_memory_pool{pool="CodeHeap 'non-profiled nmethods'",type="used",} 3188864.0
runtime_jvm_memory_pool{pool="G1 Eden Space",type="used",} 1.86646528E8
runtime_jvm_memory_pool{pool="Metaspace",type="used",} 4.5647568E7
# HELP runtime_jvm_memory_area Bytes of a given JVM memory area.
# TYPE runtime_jvm_memory_area gauge
runtime_jvm_memory_area{area="heap",type="max",} 8.34666496E9
runtime_jvm_memory_area{area="heap",type="used",} 2.04801464E8
runtime_jvm_memory_area{area="non_heap",type="committed",} 7.7848576E7
runtime_jvm_memory_area{area="heap",type="committed",} 5.22190848E8
runtime_jvm_memory_area{area="non_heap",type="used",} 7.378148E7
# HELP runtime_jvm_gc_count The number of collections that have occurred for a given JVM garbage collector.
# TYPE runtime_jvm_gc_count gauge
runtime_jvm_gc_count{gc="G1 Old Generation",} 0.0
runtime_jvm_gc_count{gc="G1 Young Generation",} 7.0
# HELP runtime_jvm_gc_time Time spent in a given JVM garbage collector in milliseconds.
# TYPE runtime_jvm_gc_time gauge
runtime_jvm_gc_time{gc="G1 Old Generation",} 0.0
runtime_jvm_gc_time{gc="G1 Young Generation",} 145.0
# HELP queueSize The number of spans queued
# TYPE queueSize gauge
queueSize{spanProcessorType="BatchSpanProcessor",} 0.0
```

Co wiecej, istnieje możliwość udostępnienia dodatkowych metryk do zebrania przez Prometheusa. Nie jest to na ten moment w żaden sposób udokumentowane ([https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/1566](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/1566)) jednak przepis na to wydaje się całkiem prosty. Należy dodać dwie biblioteki do pom.xml (lub odpowiedniego pliku dla Gradle) i zrestartować aplikacje. 

```bash
<dependency>
			<groupId>io.opentelemetry.instrumentation</groupId>
			<artifactId>opentelemetry-oshi</artifactId>
			<version>1.6.2-alpha</version>
			<scope>runtime</scope>
</dependency>
<dependency>
			<groupId>com.github.oshi</groupId>
			<artifactId>oshi-core</artifactId>
			<version>5.8.2</version>
</dependency>
```

Po uruchomieniu widać na pierwszy rzut oka, że endpoint metrics zwraca wiecej metryk jednak nie znalazłem opcji, która byłaby odpowiedzialna za kontrole co jest pobierane z biblioteki OSHI. Nie można też jednoznacznie ocenic czy jest to na minus, że trzeba dodać te dwie zależności do classpath aplikacji. Na pewno nie wszyscy bedą chcieli korzystać z exportera dla Prometheusa i wtedy dla nich nie miałoby sensu pakowanie tych dwóch jarów od razu do eksportera. 

Tym sposobem zakończyłem pierwszą cześć - konfiguracje agenta Open Telemetry do wystawianie metryk dla Prometheus. 

Kolejnym punktem na naszej liście jest pobranie Prometheusa i jego konfiguracja. Jest na to kilka sposobów:

- Można pobrać Prometheusa bezpośrednio ze strony [https://prometheus.io/download/](https://prometheus.io/download/)
- Można użyć obrazu kilku obrazów dockera (Prometheus, Grafana, Alertmanager) połącznych za pomocą docker-compose.yml w jedno środowisko. W tym przypadku, najlepiej byłoby zrobić tak, że nasz aplikacja też podpięta pod ten docker-compose.yml.

Ja zdecydowałem się na opcje pierwszą aby jak najszybciej uzyskać połączenie pomiedzy agentem a Prometheusem. Po pobraniu binarek ze strony, należy je skonfigurować naszą usługę tak aby pobierała informacje od agenta. Można to zrobić w pliku prometheus.yml gdzie należy zmodyfikować sekcje *scrape_configs* gdzie należy dodać nowy job_name. Wygląda to następująco:

```bash
- job_name: "service-a"
    static_configs:
      - targets: ["localhost:9464"]
```

Po wystartowaniu, w UI można zobaczyć, że udało się nawiązać połączenie pomiedzy Prometheusem a naszym agentem open telemetry. 

![Untitled](/images/OpenTelemetry/3/prometheus.png)

Dzięki temu połączeniu, można zacząć monitorować dane z naszego eksportera. 

![Untitled](/images/OpenTelemetry/3/exporter-data.png)

Gdy zależy nam na dobrym zwizualizowaniu danych i stworzeniu dashboardów należy użyć Grafny, której jako źródło danych można ustawić Prometheusa. 

Na samym końcu obiecalem przyjrzec sie jeszcze jakie są jeszcze inne sposoby integracji z Prometheusem i Jaegerem. 

Sam Spring Boot oferuje ujednolicony sposób zbierania i wystawiania metryk przez Actuatora. Żeby z tego skorzystać należy dodać zależność do naszego pom.xml (lub do odopowiedniego pliku dla Gradle)

```bash
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Od tego momentu, niektóre z metryk naszej aplikacji będą dostępne pod adresem [http://localhost:8080/actuator](http://localhost:8080/actuator) (po wejściu należy wybrać jakie metryki nas interesują). Actuator w ramach swoich zależności zawiera już Micrometer który posłuży nam jako baza do dodania wsparcia dla Prometheusa. Teraz należy dodać poniższą zależność, ustawić aby endpoint /prometheus był udostępniany przez Actuator i gotowe

 

```bash
<dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-registry-prometheus</artifactId>
      <scope>runtime</scope>
</dependency>
```

W przypadku Jaegera sytuacja wygląda inaczej, ponieważ nie ma integracji z Actuatorem a także nie znalazłem żadnego startera na stronie [http://start.spring.io](http://start.spring.io), który mógłby posłużyć do tego. Jednak na Githubie jest projekt, które taką integracje zdecydowanie uprości i sprowadzi do konfiguracji propertisów - [https://github.com/opentracing-contrib/java-spring-jaeger](https://github.com/opentracing-contrib/java-spring-jaeger)

Co wiecej, warto tutaj nadmienić, że java-spring-jaeger korzysta z API dostarczonego razem z projektem OpenTracing. Jest to o tyle ciekawa sytuacja, że projekt OpenTelemetry powstał z połączenia OpenTracing i OpenCensus na co wskazuje sama strona OpenTracing.

![Untitled](/images/OpenTelemetry/3/opentelemetry-merge.png)

Jak widać, w przypadku Prometheusa i Jaegera istnieją także inne możliwości integracji z usługami i agent nie jest jedynym rozwiązaniem. 

## Podsumowanie

Open tracing agent jest na pewno ciekawym projektem, który można rozważyć przy implementacji wzorca obserwowalności w swojej architekture. Na pewno na plus dla tego podejścia moża zaliczyć to, że nie wspiera on jedynie Jaegera czy Prometheusa, a posiada wiecej eksporterów. Co wiecej, jeden agent wystarczy aby różne z tych wzorców zaimplementować od razu. Actuator i startery dla Spring Boota wydają się ciekawą alternatywą, zdecydowanie bardziej natywnym doświadczeniem, który też nie jest trudno skonfigurować dla poszczególnych usług. Na minus na pewno można zaliczyć to, że do każdej usługi trzeba dodać osobną zależnosć (starter, jara) co może wymagać przebudowania projektu. Open Tracing Agent można zintegrować z istniejącą aplikacją bez zmiany kodu źródłowego na środowisku produkcyjnym co według mnie jest dużą zaletą. 

## Źródła

[https://www.jaegertracing.io/docs/1.26/architecture/](https://www.jaegertracing.io/docs/1.26/architecture/)