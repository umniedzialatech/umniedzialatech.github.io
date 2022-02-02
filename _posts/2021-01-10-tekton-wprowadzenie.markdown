---
layout: post
title:  "Tekton - Cloud Native CI/CD"
date:   2022-01-10 13:32:49 +0100
categories: tekton
---
{% include toc %}
Tekton jest to narzędzie pozwalające na tworzenie pipelinów w środowisku cloud-native. Tekton funkcjonuje w środowisku Kubernetesowym. Bazuje na Kubernetes Custom Resource czyli na funkcjonalności pozwalającej na rozszerzenie możliwości istniejącego już Kubernetes API.

# Co jest największą zaletą Tekton?

**Reużywalność.** Tekton pozwala na tworzenie elementów które można dowolnie łączyć w ramach pipelinów.  Dzięki temu można w łatwy sposób szybko zbudować skomplikowany proces bez konieczności wynajdywania koła na nowo i budowania wszystkiego od zera.

**Rozszerzalność.** Tekton cechuje się społecznością która z sukcesem tworzy kolejne rozszerzenia dzięki którym można przyspieszyć  prace używając już gotowych komponentów. Wszystko jest dostępne publicznie na [https://hub.tekton.dev/](https://hub.tekton.dev/)

**Standaryzacja**. Tekton funkcjonuje jako rozszerzenie Kubernetosowe, więc może działać w ramach istniejącego klastra.

**Skalowalność.** By zwiększyć możliwości robocze Tektona konieczne jest jedynie dodanie kolejnych nodów do klastra. Żadna dodatkowa operacja nie jest wymagana.

![Gitlab](/images/Tekton/Tekton1/gitlab1.png)

A jak widać powyżej, ta skalowalność nie jest bez znaczenia.

# Czy Tekton jest dla mnie?

Jak zawsze odpowiedź trzeba zacząć od "to zależy".

Jeśli organizacja posiada wdrożonego Kubernetesa i wszelkie repozytoria kodu nie są hostowane na np. GitLab,  Github .etc wtedy Tekton może być (ale wcale nie musi, o czym dalej) dobrym rozwiązaniem. Jeśli jednak mówimy o repozytoriach które są trzymane na GitLab, GitHub wtedy jednak zwróciłbym uwagę na:

- [GitLab CI](https://docs.gitlab.com/ee/ci/index.html)
- [Github Actions](https://github.com/features/actions)
- [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines)

Ponieważ to one w mojej opinii mogą przynieść najlepszy developer experience zwłaszcza w kombinacji **repozytorium kodu + CI/CD** w ramach tej samej platformy.

Przygotowując się do pisania tego artykułu, starałem się dowiedzieć jak najwięcej na temat Tekton i spotkałem kilka opinii nt. tego narzędzia, że jest ono bardzo elastyczne i pozwala na stworzenie wielu rozwiązań ale przy tym jest ono skomplikowane, co może odstraszyć i raczej odradza się wdrażanie samego Tektona.



![Źródło: octopus.com](/images/Tekton/Tekton1/octopus.png)

Źródło: octopus.com

### Gdzie można zobaczyć Tekton?

Tekton jest obecny w kilku produktach takich jak:

- [OpenShift Pipelines](https://cloud.redhat.com/blog/introducing-openshift-pipelines)
- [IBM Cloud](https://www.ibm.com/cloud/tekton)
- [Jenkins X](https://jenkins-x.io/)
- [Google Cloud](https://cloud.google.com/tekton)

Jak widać szeroka adaptacja tego projektu open source może dawać do zrozumienia, że w przyszłości możliwe jest, że stanie się standardem CI/CD w obszarze dostarczania oprogramowania. Warto nadmienić, że GitLab CI również rozważa umożliwienie tworzenia w ich środowisku tworzenie rozwiązań CI/CD z wykorzystaniem Tekton.  Zachęcam do przeczytania wątku na gitlab issues [Support Tekton pipeline definitions in GitLab](https://gitlab.com/gitlab-org/gitlab/-/issues/213360), w wątku pojawia się również Bitbucket jako konkurencja, może właśnie jesteśmy świadkami "wyścigu" kto pierwszy dostarczy ten feature ?

### Czym więc jest Tekton?

Z mojej perspektywy jest to próba standaryzacji całego procesu tworzenia CI/CD co pozwoli na łatwiejsze i szybsze budowanie, dostarczanie oprogramowania. Biorąc pod uwagę, że największe firmy wyrażają swoje zainteresowanie może ma to szanse powodzenia?  Może budowanie pipelinów .etc ograniczy się do wykorzystania gotowych narzędzi które wszystko zrobią niemalże same?

# Instalacja

Instalacja jest dość prosta - wystarczy posiadać działający klaster Kubernetes. Na cele tego artykułu skorzystam z możliwości bardzo prostego postawienia klastra Kubernetesowego z wykorzystaniem [Docker Desktop](https://www.docker.com/products/docker-desktop).

## Tekton Pipelines

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.29.0/release.yaml
```

Po instalacji komenda:

```bash
kubectl get namespace
```

Powinna pokazać na liście **tekton-pipelines**

![Tekton Pipelines](/images/Tekton/Tekton1/tekton-pipelines.png)

Dodatkowo możemy zweryfikować czy pody są w statusie **Running**

```bash
kubectl get pods --namespace tekton-pipelines
```

## CLI

By móc pracować z **Tekton** niezbędna będzie również instalacja CLI.

### Windows

Dla tego systemu może się to ograniczyć do skorzystania z repozytorium **Chocolatey**

```bash
choco install tektoncd-cli --confirm
```

Lub w sposób tradycyjny ściągnąć ze strony [plik](https://github.com/tektoncd/cli/releases) .zip i dodać zawartość do zmiennej środowiskowej.

### MacOS

Tutaj przydatne będzie brew.

```bash
brew tap tektoncd/tools
brew install tektoncd/tools/tektoncd-cli
```

### Linux

CLI jest również dostępne dla dystrybucji Linuxa, a pakiety instalacyjne dostępne są w następującym formacie:

- .deb
- .rpm

Dokładną instrukcje jak można zainstalować CLI znajduje się na oficjalnej stronie [tutaj](https://tekton.dev/docs/cli/#:~:text=WINDOWS-,LINUX,-tkn%20is%20available)

Teraz mamy już wszystko gotowe by móc eksperymentować z **Tekton!**

# Pierwszy Task

Teraz pozostaje tylko sprawdzić, czy wszystko działa, w tym celu utworzymy pierwszy prosty **Task.**

```yml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: hello
      image: ubuntu
      command:
        - echo
      args:
        - "Hello World!"
```

Powyższy fragment kodu zapisujemy jako plik **task-hello.yaml** a następnie tworzymy zasób wykonując

```bash
kubectl apply -f task-hello.yml
```

Utworzony został nowy zasób o nazwie **hello** by to zweryfikować możemy wykonać

```bash
kubectl get tasks
```

![Tasks](/images/Tekton/Tekton1/tasks.png)

Teraz pozostaje tylko uruchomić **Task**

```bash
tkn task start hello
```

By potwierdzić, że wszystko się udało, musimy sprawdzić logi:

```bash
tkn taskrun logs --last -f
```

Rezultat

![Hello](/images/Tekton/Tekton1/hello.png)

Jak widać, wszystko działa!

# Podsumowanie

W tym artykule dowiedzieliśmy się kilku podstawowych informacji nt. Tekton takich jak:

- Czym jest Tekton
- Jak zainstalować Tekton Pipelines
- Jak zainstalować CLI
- Jak uruchomić prosty Task

W kolejnym artykule poznamy:

- Podstawowe zasoby dostępne w ramach Tekton Pipelines
- Stworzymy pierwszy **Tekton Pipeline**
- Zainstalujemy dodatkowe komponenty

Kod **task-hello.yml** dostępny jest również na [GitHub](https://github.com/umniedzialatech/tekton/tree/master/tekton1).

## Źródła

- [https://tekton.dev/](https://tekton.dev/)
- [https://hub.tekton.dev/](https://hub.tekton.dev/)