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
kubectl apply --filename hhttps://github.com/tektoncd/dashboard/releases/download/v0.26.0/tekton-dashboard-release.yaml
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

- **TasksRuns** - wyświetla informacje o uruchomionych taskach

![TasksRuns](/images/Tekton/3/tasksruns.png)
Po wejściu w daną instancje uruchomienia widoczne są szczegółowe informacje
![TasksDetails](/images/Tekton/3/tasksrunsdetails.png)
W ten sposób poznaliśmy podstawowe możliwość dostępne w Tekton Dashboard, ale jest jedna rzecz której mi osobiście brakuje - wizualizacji przepływów.

# Namespace
Uważne oko dostrzeże również w prawym górnym rogu możliwość wyboru **namespace**, czyli logicznego obszaru roboczego.<br>
Domyślnie wybrany jest obszar “default” ale istnieje możliwość stworzenia kompletnie innego. <br>Dlaczego jest to ważne?<br>
Pozwala to między innymi na, stworzenie namespace dla każdego środowiska (dev, test .etc).<br>
Dodatkowo umożliwia również zaaplikowanie limitów związanych z zasobami np. Pody w danym namespace nie mogą mieć przydzielone więcej niż 256 MB pamięci.
# ClusterTask vs Task
Różnica jest bardzo prosta:

**ClusterTask** - jest dostępny w ramach całego klastra

**Task** - jest dostępny tylko w ramach danego namespace w którym został stworzony

Do czego potencjalnie może to być użyteczne?

Może ułatwić zarządzanie podstawowymi Taskami takimi jak np. git-clone.  <br>Gdy będzie wymagał aktualizacji odbędzie się ona w jednym miejscu co znacząco uprości i przyśpieszy proces.
# Wizualizacja przepływów 

Niestety nie udało mi się znaleźć tej funkcjonalności w Tekton Dashboard  a osobiście uważam ją za bardzo pomocną w zrozumieniu samego działania mechanizmu.

Utworzone zostało [Github Issue](https://github.com/tektoncd/dashboard/issues/675), ostatnia aktywność jest z dnia 24.04.2022.<br>
Została tam wymieniona min. funkcja dostepna w Tekton Pipelines by Red Hat - wizualizacja pipelineów.
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

