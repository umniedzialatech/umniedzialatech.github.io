---
layout: post
title:  "[2] Tekton - Pierwszy Pipeline"
date:   2022-02-10 13:32:49 +0100
categories: tekton
---
{% include toc %}
W poprzednim artykule udało się zainstalować Tekton, uruchomić pierwszy Task.
Teraz poznamy więcej dostępnych elementów.

# Podstawowe elementy

## Tekton Pipelines

Elementy typu Pipeline w Tektonie opierają się na trzech głównych składowych:

- **Step.** Jest to krok wykonywalny w procesie CI/CD, może to być np. uruchomienie testów, kompilacja kodu. W poprzednim artykule takim krokiem był element **hello** zdefiniowany w **task-hello.yml** odpowiedzialny za powitanie świata.
- **Task.** Jest to zbiór kroków (steps) które są uruchamiane w ustalonej kolejności w postaci podów Kubernetesowych.
- **Pipeline.** Jest to kolekcja tasków w ustalonej kolejności.


Oficjalna dokumentacja Tekton w obrazuje to w bardzo czytelny sposób na poniższej grafice.

![Obrazowe przedstawienie podstawowych elementów Tekton. Źródło: tekton.dev](/images/Tekton/2/pipeline.png)
*Obrazowe przedstawienie podstawowych elementów Tekton. Źródło: tekton.dev*

## Input/Output

Każdy **task** oraz **pipeline** może posiadać własne parametry wejściowe oraz wyjściowe. Dobrym przykładem może być **compilation-task** który może przyjąć repozytorium git a zwrócić skompilowany kod.

## TaskRuns/PipelineRuns

Są to zasoby Kubernetesowe, które reprezentują instancje, konkretnego uruchomienia Taska lub Pipeline. Dla przykładu w poprzednim artykule uruchomiliśmy **hello-task** i ta informacja jest dostępna po wykonaniu

```bash
kubectl get taskruns
```

![Untitled](/images/Tekton/2/taskruns.png)

Poniżej zamieszczam tabelaryczne zestawienie wszystkich elementów dostępnych w Tekton.

![Elementy dostępne w Tekton. Źródło: tekton.dev](/images/Tekton/2/tektontable.png)
*Elementy dostępne w Tekton. Źródło: tekton.dev*

# Budowa

Teraz gdy już mamy wszystkie niezbędne informacje, pozostaje tylko zacząć budować pierwszy funkcjonalny Pipeline. Założenie jest proste:

- Pobranie kodu z repozytorium git
- Uruchomienie testów i kompilacja

Wraz z kolejnymi artykułami postaram się rozbudować ten pipeline o bardziej skomplikowane/zbliżone do realnych projektów kroki.

## Git-clone

### Instalacja

Pierwszym krokiem jest sklonowanie repozytorium, w tym celu skorzystamy z gotowego pluginu [git clone](https://hub.tekton.dev/tekton/task/git-clone).

Instalacja jest prosta, polega na uruchomieniu komendy:

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.5/git-clone.yaml
```

Następnie możemy zweryfikować czy plugin został poprawnie zainstalowany, wykonując:

```bash
kubectl get tasks
```

Na liście powinnien być obecny plugin git-clone:

![Untitled](/images/Tekton/2/gitclone.png)

### Wykorzystanie

Teraz możemy zacząć budować pipeline.

Początek będzie wyglądał następująco:

```bash
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: app-deploy1
spec:
  workspaces:
    - name: shared-workspace 
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
      params:
        - name: url 
          value: https://github.com/spring-petclinic/spring-petclinic-rest.git
      workspaces:
      - name: output
        workspace: shared-workspace
```

Jak widać, użycie pluginu wygląda dość prosto, podajemy dwa parametry:

- adres repozytorium git
- workspace

O ile ten pierwszy jest dość oczywisty to drugi, workspace jest ciekawszy.

### Czym jest workspace?

Workspace to wspólna przestrzeń pozwalająca na dzielenie zasobów systemowych, plików pomiędzy poszczególnymi krokami w pipelinie.

W powyższym przykładzie repozytorium zostanie sklonowane do workspace'u **shared-workspace.**

### Jak stworzyć workspace?

Definicja repozytorium wygląda następująco:

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
```

By je utworzyc wystarczy wykonać poniższe polecenie wskazując plik zawierający powyższa definicje.

```bash
kubectl apply -f shared-workspace.yml
```

Rezultat:

![Untitled](/images/Tekton/2/result.png)

Najciekawszą częścią z powyższego kodu jest fragment związany z **accessModes**, obecnie Tekton wspiera następujące ustawienia:

- **ReadWriteOnce** - cechą charakterystyczną tego trybu jest fakt, że ten zasób może być zamontowany tylko raz na danym węźle. Może stanowić to problem dla elementów typu **Task** które wykonują się równocześnie i potrzebują mieć dostęp do tego zasobu. Na szczęście **Affinity Assistant** pozwala rozwiązać ten problem przez przypisanie wszystkich Tasków do które korzystają z tego samego **PersistentVolumeClaim** do tego samego węzła.
- **ReadOnlyMany** - cechuje się tym, że pozwala tylko wyłącznie na **odczyt** przez wiele węzłów. Zwykle jest przygotowywany ręcznie w celu wykorzystania. todo przykład np. credentails
- **ReadWriteMany** - sprawia, że zasób jest dostępny do zapisu i odczytu dla wszystkich węzłów w klastrze Kubernetesowym.

Teraz gdy już mamy repozytorium dostępne pora na kolejny krok.

## Maven

W zależności od projektu możemy potrzebować różnych narzędzi - w tym przypadku do zbudowania paczki wdrożeniowej konieczne będzie wykonanie kilku Goali MVN.

I tym razem dostępny jest plugin na [hub.tekton.dev](http://hub.tekton.dev) → [maven](https://hub.tekton.dev/tekton/task/maven).

**Instalacja**

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/maven/0.2/maven.yaml
```

Rezultat:

![Untitled](/images/Tekton/2/resultmaven.png)

Teraz pozostaje dodać odpowiednie krok do pipeline.

```bash
- name: maven-run
      taskRef:
        name: maven
      runAfter:
        - git-clone
      params:
        - name: CONTEXT_DIR
          value: "./"
        - name: GOALS
          value:
            - -DskipTests
            - clean
            - package
      workspaces:
      - name: source
        workspace: shared-workspace 
      - name: maven-settings
        workspace: maven-settings
```

Jak widać ten fragment nie jest bardzo skomplikowany, zawiera jedno nowe wyrażenie kluczowe

**runAfter**.

Jak łatwo się domyślić słowo to jest odpowiedzialne za kolejność wykonywania się poszczególnych kroków w trakcie wykonywania pipeline.

## Ostatni krok - wyświetlenie wyniku

Ostatnim krokiem w pipeline, będzie weryfikacja czy projekt zbudował się poprawnie i czy plik uruchomieniowy z rozszerzeniem **.jar** jest dostępny.

Możemy to zrobić dodając kolejny krok.

By to zrobić, stworzymy definicje Taska, **list-directory.**

```bash
kubectl apply -f list-directory.yml
```

Ostateczny pipeline będzie wyglądał następująco:

```bash
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: app-deploy1
spec:
  workspaces:
    - name: shared-workspace 
      description: |
        This workspace is shared among all the pipeline tasks to read/write common resources  
    - name: maven-settings
    - name: directory
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
      params:
        - name: url 
          value: https://github.com/spring-petclinic/spring-petclinic-rest.git
      workspaces:
      - name: output
        workspace: shared-workspace 
    - name: maven-run
      taskRef:
        name: maven
      runAfter:
        - git-clone
      params:
        - name: CONTEXT_DIR
          value: "./"
        - name: GOALS
          value:
            - clean
            - package
      workspaces:
      - name: source
        workspace: shared-workspace 
      - name: maven-settings
        workspace: maven-settings
    - name: show-files
      taskRef:
        name: list-directory
      runAfter:
        - maven-run
      params:
        - name: sub-dirs
          value: [./target]
      workspaces:
      - name: directory
        workspace: shared-workspace
```

Tworzymy go wykonując:

```bash
kubectl apply -f first-pipeline.yml
```

Rezultat:

![Untitled](/images/Tekton/2/resultpipeline.png)

Teraz pozostaje tylko sprawdzić działanie.

## Uruchomienie

Ostatnim krokiem będzie uruchomienie całego rozwiązania.

```bash
tkn pipeline start app-deploy1 \
  --workspace name=shared-workspace,subPath=,claimName=shared-pvc \
  --workspace name=maven-settings,emptyDir="" \
  --workspace name=directory,subPath=,claimName=shared-pvc \
  --use-param-defaults \
  --showlog
```

Pwyżej musimy przekazać 3 workspace:

- **shared-workspace** - przestrzeń która pozwala na przekazywanie plików pomiędzy poszczególnymi krokami w tym wypadku dzielone jest sklonowane repozytorium  (git-clone) z mavenem (maven)
- **directory** - przestrzeń której zawartość ma zostać wyświetlona w tym przypadku

  **shared-workspace** gdzie znajdziemy skompilowany kod

- **maven-settings** -opcjonalny workspace który pozwala na przekazanie ustawień dla mavena np. adres repozytorium

## Rezultat

Jak widać poniżej projekt został poprawnie zbudowany.

![Untitled](/images/Tekton/2/resultfinal.png)

## Podsumowanie

W artykule zostały przedstawiony prosty pipeline z elementami:

- Klonowania repozytorium (git-clone)
- Wykorzystania Workspace
- Wykonywania celi zdefiniowanych w pliku pom.xml (maven)
- Wylistowanie zawartości workspace, celem sprawdzenia czy wszystko przebiegło pomyślnie

Jak widać można w dość łatwy sposób stworzyć prosty i funkcjonalny pipeline

W kolejnym artykule:

- Wykorzystanie volumeClaimTemplate
- Budowanie obrazu dockerowego
- Dodatkowe kroki w pipeline
- Wizualizacja pipeline
- Drobne optymalizacje

