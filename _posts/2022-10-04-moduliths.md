---
layout: post
title:  "[1] Moduliths - testy dla modularnego monolitu"
date:   2022-10-04 20:00:00 +0100
categories: archunit,architektura
---
{% include toc %}

W poprzednim artykule przedstawiłem narzędzie jakim jest ArchUnit. Tym razem spróbuje przybliżyć kolejną biblioteke nazwaną [Moduliths](https://github.com/moduliths/moduliths). W założeniu, te rozwiązanie ma wspierać implementacje modularnych aplikacji poprzez zgrupowanie modułów wokół pojedycznego pakietu javy.

Jak można przeczytać na Githubie biblioteki, motywacją do powstania tego rozwiązania było wyśrodkowanie podejścia jakim jest architektura mikroserwisów i monolitu. Najcześciej, te dwie architektury przedstawiane są jako przeciwne podejścia, a autor próbuje dostarczyć narzędzie, które posłuży jako pomoc przy implementacji modularnego monolitu (modulith) - czegoś po środku. 

Czym jest modularny monolit?

Zastosowanie modularnego monolitu może okazać się bardzo pomocne jako pierwszy krok w strone poprawy architektury aplikacji. Dzieki takiemu podejściu możemy już wstępnie wyznaczyć granice naszych modułów i z pomocą IDE / kompilatora przeprowadzić refaktoryzacje. Ewentualne błędy, które zostana popełnione przy dzieleniu na moduły bedą zdecydowanie prostrze w naprawie i tańsze niż jakbyśmy to robili już na mikroserwisach. Komunikacja pomiedzy modułami może odbywać się na różne sposoby, ale żeby zredukować coupling, można skorzystać z *Spring Events* - zdarzeń w pamięci. Wydzielone moduły mogą wysyłać wiadomości pomiedzy sobą korzystając z tego mechanizm. Oczywiście, można też skorzystać z komunikacji synchronicznej (poprzez zapytanie http do innego modułu) Warto tutaj zaznaczyć, że aplikacja w dalszym ciągu będzie wdrażana jako pojedyńczy artefakt, nie będzie możliwości skalowania wybranych modułów.

## Moduliths

Jak zaznacza sam [Oliver Drotbohm](https://github.com/odrotbohm), biblioteka **Moduliths** jest “placem zabaw” (a playground) do eksplorowania jak tworzyć monolityczne aplikacje. Podejście, które zostało wybrane przez autora skupia się na konwencji budowaniu modułów opartych wokół pojedyńczego pakietu. Bazowy pakiet jest definiowany poprzez adnotacje **@Modulith** czyli ta adnotacja w naszym przykładzie który zaczniemy teraz tworzyć, będzie umieszczona w pakiecie tech.umniedziala a następnie możemy zdefiniować dwa moduły / pakiety. 

**tech.umniedziala.moduleA**

**tech.umniedziala.moduleB** 

t**ech.umniedziala.moduleC**

t**ech.umniedziala.moduleD**

Od razu - równocześnie - powinniśmy stworzyć testy dla tych modułów aby upewnić, że spełniamy wszystkie wymagania. Biblioteka sprawdza:

- Cykliczne zależności pomiedzy modułami (klasami)
- Czy którykolwiek z modułów nie polega na wewnetrznym komponencie innego modułu
- Czy przestrzegane sa zasady Ddd dostarczone przez jmolecules. (jeżeli biblioteka jest na classpath)

Wszystko to można sprawdzić w klasie [Modules.java](http://Modules.java) w metodzie detectViolations.

![1](/images/Archunit/2/1.png)

Jak stworzymy testy dla modułu A i opakujemy je adnotacją @ModuleTest, to od razu w logach widać, że będziemy mieli możliwość testować jedynie komponenty modułu A. 

```java
@ModuleTest
public class ModuleATests {

    @Autowired
    private InternalComponentA internalComponentA;

    @Test
    void moduleTest() {
        // intentionally left empty
    }
}
```

 I logi po starcie testu. 

```java
2022-05-21 17:31:29.452  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : Bootstrapping @ModuleTest for tech.umniedziala.moduleA in mode STANDALONE (class tech.umniedziala.TestApplication)…
2022-05-21 17:31:29.453  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : ===================================================================================================================
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : ## tech.umniedziala.moduleA ##
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : > Logical name: moduleA
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : > Base package: tech.umniedziala.moduleA
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : > Direct module dependencies: none
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer : > Spring beans:
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer :       + ….ComponentA
2022-05-21 17:31:29.457  INFO 17287 --- [           main] ustomizerFactory$ModuleContextCustomizer :       + ….internal.InternalComponentA
2022-05-21 17:31:29.463  INFO 17287 --- [           main] tech.umniedziala.moduleA.ModuleATests    : Starting ModuleATests using Java 17.0.2 on H
```

Test przeszedł w porządku, żaden błąd nie został zgłoszony.  Jednak jeżeli tylko w testach dla modułu A dodamy, że chcemy skorzystać z *ComponentB* (który jest w module B), to ten test od razu zakończy sie niepowodzeniem z błędem, że nie można znaleźć takiego bean’a (@ModuleTest domyślnie iniciuje zależności jedynie które znajduja się w module A) 

### Cykliczna zależność miedzy modułem C a D

Kolejną weryfikacją jaką możemy przeprowadzić jest sprawdzenie cyklicznej zależności. Jeżeli stworzymy cykliczną zależność pomiedzy modułem C i D, to uruchomienie testów dla modułu C (lub D, nie ma różnicy) sprawi, że po wykonaniu testu ujrzymy następujący błąd.

```java
Caused by: org.moduliths.model.Violations: - Cycle detected: Slice moduleC -> 
                Slice moduleD -> 
                Slice moduleC
```

## Sprawdzanie zasad dostarczonych przez jmolecules

Najpierw zajme się ustaleniem czym jest biblioteka jmolecules. (Przed rozpoczeciem research’u na temat Moduliths nie miałem wiedzy, że takie narzędzie istnieje) Jak mówi sam autor (znowu [Oliver Drotbohm](https://github.com/odrotbohm)) na githubie,  https://github.com/xmolecules/jmolecules, biblioteka służy do tego aby uprościć wyrażanie koncepcji archtekturalnych w kodzie. Realizowane to jest poprzez adnotacje lub interfejsy. 

Dla przykładu - w przypadku DDD -  biblioteka oferuje miedzy innymi  takie adnotacje jak @Entity, @ValueObject czy @Repository. Co warto wspomnieć, to że biblioteka służy nie tylko do DDD i posiada też odpowiednie implementacje dla takich podejści jak CQRS, onion architecture, hexagonal czy dobrze znana layered archtecture. 

Wiecej o tym narzędziu można się dowiedzieć [w tym wystapieniu na Youtube](https://www.youtube.com/watch?v=Q_B1swi9_N8)

Teraz sprawdźmy integracje, która występuje pomiedzy Moduliths, a jMolecules. Uważni czytelnicy mogli zauważyć, że w poprzednim zdjęciu, na którym pokazywałem kod wykonujący walidacje, brakowało zależności do klasy JMoleculesDddRules, teraz jednak ją dodamy w pom.xml poprzez zależność:

```xml
<dependency>
    <groupId>org.jmolecules.integrations</groupId>
    <artifactId>jmolecules-archunit</artifactId>
    <version>0.10.0</version> <!-- warto sprawdzić jaka wersja aktualnie jest najnowsza -->
</dependency>
```

Gdy to mamy, w ramach testów które wykonuje biblioteka Modulthis otrzymujemy także następujące sprawdzenia (po angielsku): 

- fields that are declared within a jMolecules domain type and are assignable to org.jmolecules.ddd.types.Entity and not are assignable to org.jmolecules.ddd.types.AggregateRoot should belong to aggregate the field is declared in
- fields that are declared within a jMolecules domain type and are assignable to org.jmolecules.ddd.types.AggregateRoot or is of type annotated with AggregateRoot should use id reference or Association
- classes that annotated with @AggregateRoot or annotated with @Entity and are not annotations should declares field (meta-)annotated with org.jmolecules.ddd.annotation.Identity

Wszystkie walidacje bazują na adnotacjach z biblioteki jmolecules-ddd. 

## Generowanie diagramu

Ostatnią możliwościa, którą chciałem sprawdzić w tej bibliotece jest generowanie diagramów na podstawie kodu. Do tego będzie potrzebna nowa zależność:

```xml
<dependency>
    <groupId>org.moduliths</groupId>
    <artifactId>moduliths-docs</artifactId>
    <version>${modulith.version}</version>
    <scope>test</scope>
</dependency>
```

Następnie z pomocą takiego kodu będzie można wygenerować digramy dla całej naszej aplikacji.

```java
Modules modules = Modules.of(TestApplication.class);
Documenter documenter = new Documenter(modules);
Documenter.Options options = Documenter.Options.defaults();
// Write overall diagram
documenter.writeModulesAsPlantUml(options);
// Write module canvases
documenter.writeModuleCanvases(Documenter.CanvasOptions.defaults().withApiBase("{javadoc}"));
```

API Documenter’a oferuje kilka opcji do konfiguracji w jaki sposób te moduły mogą być generowane. Od takiej standardowej konfiguracji jak zmiana nazwy pliku do takich detali jak styl generowanego diagramu (oprócz UML, można też wygenerować diagram w stylu C4). Ja zdecydowalem się na jedną zmiane konfiguracji: 

```java
Documenter.Options options = Documenter.Options.defaults()
                .withElementsWithoutRelationships(VISIBLE);
```

Domyślnie, Documenter nie pokazuje na wygenerowanym diagramie elementów bez relacji. 

![2(/images/Archunit/2/2.png)

Dodatkowo, dla każdego modułu można wygenerować adoc z informacjami jakie komponenty Springa tam są zarejestrowane, jakie eventy są generowane - wszystko jest mocno osadzone w ekosystemie Springa. Przykładowo dla modułu A wygląda to w następujący sposób

![3](/images/Archunit/2/3.png)

## Podsumowanie

Moduliths na pewno wydaje się bardzo ciekawym narzędziem, szczególnie dla użytkowników Springa którzy zastanawiają się jak zaimplementować modularny monolith. Warto jednak zwrócić tutaj uwage, że przedstawiona metoda implementacji jest bardziej opinią autora w jaki sposob można implementować modularny monolith i stworzył on na bazie ArchUnit narzędzie do walidacji jego podejścia. Nie wszystkim takie coś może odpowiadać (niektórzy zamiast pakietów mogą zdecydować się na podział modułów z pomocą narzędzia do budowania). Co według mnie jest na minus to wybrakowana dokumentacja, gdybym nie sprawdził kodu biblioteki, nie wiedziałbym, że istnieje integracja pomiedzy Moduliths a jMolecules. Niemniej jednak, koncepcje takie jak przesyłanie zdarzeń pomiedzy modułami w ramach jednego procesu jvm, wydaje się dobrą drogą aby rozpocząć podział swojego monolitu na mniejsze fragmenty. JWarto nadmienic, że jest to przykład ciekawego zastosowania ArchUnit i właściwie każda bibiloteka mogłaby spróbować udostepnic zestaw dobrych praktyk, jak z niej korzystać, zapisanych w ArchUnit, a następnie udostępnić takie sprawdzenia jako osobny artefakt.

Repozytorium:  [https://github.com/umniedzialatech/moduliths](https://github.com/umniedzialatech/moduliths)

### Źródła

[https://github.com/moduliths/moduliths](https://github.com/moduliths/moduliths) - repozytorium biblioteki. Autor opisuje w niej główne założenia i podaje prostrze przykłady.

[https://www.youtube.com/watch?v=bVaiTPYlHFE](https://www.youtube.com/watch?v=bVaiTPYlHFE) - prezentacja o implementacji modularnego monolitu

[https://www.youtube.com/watch?v=YYvc-DNuwr8](https://www.youtube.com/watch?v=YYvc-DNuwr8) - prezentacja o modularności.