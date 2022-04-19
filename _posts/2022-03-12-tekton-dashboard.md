---
layout: post
title:  "[3] Tekton - Dashboard"
date:   2022-03-12 13:32:49 +0100
categories: tekton
---
{% include toc %}
W tym artykule przedstawię jakie możliwości daje Tekton, jeśli chodzi o szeroko rozumiane dashboardy.

## Instalacja - Tekton Dashboard

By Tekton Dashboard był dostępny konieczne jest uruchomienie kilku prostych komend, dostępnych [tutaj](https://tekton.dev/docs/dashboard/#installation).

Pierwszą z nich jest:

```bash
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml
```

Po chwili możemy potwierdzić, czy wszystko zostało zainstalowane uruchamiając kolejną komendę (wszystkie elementy powinny mieć status **Running**)**.**

```bash
kubectl get pods --namespace tekton-pipelines
```

![TektonPipeline](/images/Tekton/3/tektonpipeline.png)

Następnym krokiem jest udostępnienie dashboardu następującą komendą:



```bash
kubectl proxy --port=8080
```

Teraz możemy otworzyć - [link](http://localhost:8080/api/v1/namespaces/tekton-pipelines/services/tekton-dashboard:http/proxy/)  i powinnien się pojawić dashboard

# Przegląd dostępnych funkcji

Tekton Dashboard posiada opcje przeglądu w podstawowej wersji następujących elementów:

- **Pipelines** - wyświetla wszystkie utworzone pipeline w ramach Tekton

![Pipeline](/images/Tekton/3/pipeline.png)

- **PipelineRuns** - wyświetla informacje o uruchomionych pipelineach

![PipelineRuns](/images/Tekton/3/pipelineruns.png)

- **Tasks** - prezentuje dostępne taski które mogą zostać użyte do tworzenia pipelinów lub uruchomione jako niezależne elementy.

![Tasks](/images/Tekton/3/tasks.png)

- **TaskRuns** - wyświetla informacje o uruchomionych taskach

![TasksRuns](/images/Tekton/3/tasksruns.png)

W ten sposób poznaliśmy podstawowe możliwość dostępne w Tekton Dashboard, ale jest jedna rzecz której mi osobiście brakuje - wizualizacji przepływów.

# Wizualizacja przepływów 

Niestety nie udało mi się znaleźć tej funkcjonalności w Tekton Dashboard  a osobiście uważam ją za bardzo pomocną w zrozumieniu samego działania mechanizmu.

Utworzone zostało [Github Issue](https://github.com/tektoncd/dashboard/issues/675), ostatnia aktywność jest z dnia 22.01.2021.

## **Tekton Pipelines by Red Hat**

Jest to narzędzie które pozwala na wizualizacje min. pipelineów.

![RedHat](/images/Tekton/3/redhat.png)
*Pipeline stworzony w poprzednim artykule*

Dodatkowo z poziomu wtyczki jest możliwy dostęp do:
- Tasks (definicja Tasków)
- TaskRuns (informacje o uruchomieniu poszczególnych Tasków)
- Pipelines (definicja stworzonych pipelineów)
- PipelineRuns (informacje o stanie wykonywania)
- oraz wiele innych 


### Jak zainstalować?

Dostępnę są następujące wersje pluginów:
- [Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-tekton-pipelines)
- [IntelliJ IDEA](https://plugins.jetbrains.com/plugin/14096-tekton-pipelines-by-red-hat) 


 
## Podsumowanie

Przedstawione zostało narzędzie **Tekton Pipelines by Red Hat** które może wspomóc prace z Tektonem. 

