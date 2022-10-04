---
layout: post
title:  "[1] Archunit - przetestuja swoja architekture"
date:   2022-10-04 20:00:00 +0100
categories: archunit,architektura
---
{% include toc %}
Do zapewnienia jakości w kodzie przy którym na codzień pracujemy wykorzystuje się takie narzędzia jak SonarQube czy Checkstyle. W dużym uproszczeniu, obydwa narzędzia można uruchomić zarówno lokalnie jak i na serwerze CI, w momencie wypchnięcia zmiany na repozytorium (lub przy stworzeniu pull requestu). 

Co istotne, SonarQube czy Checkstyle oferują także możliwośc tworzenia własnych zasad (wiecej o tym można poczytać tutaj [https://docs.sonarqube.org/latest/extend/adding-coding-rules/](https://docs.sonarqube.org/latest/extend/adding-coding-rules/) jak i tutaj [https://checkstyle.sourceforge.io/writingchecks.html](https://checkstyle.sourceforge.io/writingchecks.html)) jednak w obydwóch przypadkach wydaje się to dosć skomplikowane i czasochłonne. W takiej sytuacji, można rozważyć ArchUnit, który zdecydowanie upraszcza sposób dodawania nowych zasad, które mają pilnować jakości w kodzie. Zapisuje się je w jezyku Java (lub można też w innym jezyku na platformie JVM jak Kotlin) jako test jednostkowy. Według mnie, jest to duża zaleta, ponieważ dzieki takiem podejściu można szybko dowiedzieć się czy wprowadzane zmiany nie zepsuły założeń architektonicznych projektu (szybki ‘feedback loop’), ponieważ testy archunit mogą uruchamiać się z pozostałymi testami jednostkowymi, które już są w projekcie. Poniżej pokazuje jak może wyglądać przykładowy test ArchUnit

```java
@ArchTest
public static final ArchRule rule1 = classes()
	.that().resideInAPackage("..service..")
	.should().onlyBeAccessed().byAnyPackage("..controller..", "..service..");
```

Źródło:[Przyklad z dokumentacji](https://www.archunit.org/userguide/html/000_Index.html#_using_junit_4_or_junit_5)

## DSL

Pierwsze na co można zwrócić uwage czytająć powyższy test jest łatwość z jaką się go czyta. Domain Specific Language (DSL) jest tutaj wielką zaletą ArchUnit, którą wymieniają recenzenci. Za punkt wejściowy do tego pieknego DSL można potraktować klase ArchRuleDefinition, której zawartość warto dobrze znać. 

![Java doc](/images/Archunit/1/javadoc.png)

W celu napisania własnego testu, najpierw trzeba określić co będzie testowane (może to być moduł, klasa, pole czy też konkretne metody i konstruktory) a następnie wybrany element można zawieźić do konkretnego podzbioru. 

```
Klasy, które są oznaczone adnotacją rest controller i są publiczne
```

W ArchUnit byłoby to zapisane w następujący sposób

```java
classes().that().areAnnotatedWith(RestController.class)
.and().arePublic()

```

Jak widać, nie ma problemu aby tworzyć bardziej skomplikowane przykład gdzie znajduje się wiecej niż jeden warunek. Gdy jest określony zbiór elementów na którym chcemy wykonać sprawdzenie trzeba teraz skorzystać skorzystać z *should()*  pochodzącego z *GivenClassesConjunction* który nam w tym pomorze. 

Napiszmy cały test w ten sposób:

```
Klasy, które są oznaczone adnotacją rest controller i są publiczne
powinny znajdować się w pakiecie controller.
```

Implementacja w Javie:

```java
classes().that().areAnnotatedWith(RestController.class).and().arePublic()
				 .should().resideInAPackage("..controller..");
```

Poniżej przedstawie jeszcze kilka testów z krótkim opisem co robią

```java
public static final ArchRule spring_components_should_use_only_constructor_injection = fields()
	.that().areDeclaredInClassesThat()
	.areAnnotatedWith(RestController.class)
	.or().areAnnotatedWith(Service.class).should().notBeAnnotatedWith(Autowired.class);
```

*Pola w klasach które oznaczone są adnotacja RestController i Service powinny nie być oznaczone adnotacją Autowired.*

```java
		@ArchTest
public static final ArchRule logger_fields_should_be_private_and_final = fields()
	.that().haveRawType(Logger.class).should().bePrivate()
	.andShould().beFinal().andShould().beStatic();
```

*Pola które są typu Logger powinny być prywatne, final i statyczne.* 

Te kilka wyżej wymienionych przykładów obrazuje filozofie jaka stała za powstaniem ArchUnit. Autorzy chceli dać programistom narzędzie, które umożliw w szybki sposób tworzyć testy weryfikujące poprawność architektury na poziomie aplikacji i można je wpiąć w istniejący zestaw testów. 

Co wiecej, dzialanie ArchUnit nie ogranicza sie jedynie do Javy i w miedzy innymi w Kotlinie też takie testy można tworzyć. Ponizej przykład jak to wyglada. 

```kotlin
	@ArchTest
    val 'logger field should be private and final' = fields()
            .that().haveRawType(Logger.class).should().bePrivate()
			.andShould().beFinal()
			.andShould().beStatic();
```

Zaletą jest tutaj skorzystanie z funkcjonalności jezyka Kotlin, która pozwalaja na tworzenie czytelniejszych nazw zmiennych (lub metod)

## Wprowadzenie ArchUnit do projektu

Wprowadzenie biblioteki do projektu zaczyna się od dodania zależności w Maven lub Gradle. Dla Mavena będzie to wyglądać następująco:

 

```xml
<dependency>
	<groupId>com.tngtech.archunit</groupId>
	<artifactId>archunit-junit5</artifactId>
	<version>${archunit.version}</version>
	<scope>test</scope>
</dependency>
```

Gdy jest to nowy projekt i nie posiada bagażu legacy, od razu można przystąpić do pisania nowych testów. Przykladowy szablon klasy w której umieścimy testy wygląda tak

```java
@AnalyzeClasses(packages = "tech.umniedziala")
public class ArchRules {
    @ArchTest
    public static final ArchRule first_test = null // zamiast nulla należy umieścić tutaj swój pierwszy test
}
```

W przypadku jednak gdy wprowadzamy biblioteke do projektu, który istnieje od kilkunastu lat i nie wszystkie zasady są od razu spełnione, a zależy nam jednocześnie na przestrzeganiu zasad dla nowo wprowadzonych zmian w repozytorium, to można wtedy skorzystać z FreezingArchRule.
Jest to klasa - dekorator -  wokół ArchRule, który umożliwa zapisanie aktualnego stanu naruszen i przy nastepnym starcie testu, jedynie raportowac nowe rzeczy, ktore sie pojawily nie uwzgledniajac tych zapisanych w ViolationStore (to miejsce gdzie te naruszenie są zapisywane). Wiecej o tym mechanizmie można przeczytać tutaj - [https://www.javadoc.io/doc/com.tngtech.archunit/archunit/latest/com/tngtech/archunit/library/freeze/FreezingArchRule.html](https://www.javadoc.io/doc/com.tngtech.archunit/archunit/latest/com/tngtech/archunit/library/freeze/FreezingArchRule.html)

Mechanizm ten w zapisuje w **archunit_store** (lokalizacja jest konfigurowalna) wszystkie błedy, a nastepnie przy kolejnych uruchomieniach są one filtrowane i nie raportowane do dewelopera. 

Przykladowy plik z błędem wygląda tak

```java
Field <tech.umniedziala.archunitdemo.controller.HelloController.helloRepository> 
has type <com.pz.archunitdemo.repository.HelloRepository> in (HelloController.java:0)
```

Logi dla tego scenariusza prezentuja się następująco

```
10:34:06.030 [main] INFO com.tngtech.archunit.library.freeze.ViolationStoreFactory$TextFileBasedViolationStore - Initializing TextFileBasedViolationStore at /home/zlotowski/self-dev/archunitdemo/archunit_store/stored.rules
10:34:06.080 [main] DEBUG com.tngtech.archunit.library.freeze.FreezingArchRule - No results present for rule 'Layered architecture consisting of (optional)
layer 'CONTROLLER' ('..controller..')
layer 'SERVICE' ('..service..')
layer 'REPOSITORY' ('..repository..')
where layer 'REPOSITORY' may only be accessed by layers ['SERVICE']
where layer 'SERVICE' may only be accessed by layers ['CONTROLLER']
where layer 'CONTROLLER' may not be accessed by any layer, because Our project is based on three-trier architecture'. Freezing rule result...
```

Warto tutaj podkreślic, że jeżeli naprawimy ten błąd, to zostanie on usuniety z tego pliku tekstowego, a nastepnie jezeli wprowdzimy go ponownie, to zostanie on zaraportowany jako nowy błąd - nie będzie już filtrowany.

## Praktyczny przykład - zabronienie nowych wywołań dla dopiero co oznaczonej metody deprecated

Teraz spróbujemy połączyć wiedze z dwóćh poprzednich rozdziałów i napisać test, którego głównym zadaniem będzie zabronienie wywołań metody deprecated. Istotnym założeniem, które przyjąłem jest to, że wszystkie poprzednie wywołania tej metody mi nie przeszkadzaja, chce jedynie zabronić tych wyłoań, które powstaną po wprowadzeniu adnotacji Deprecated na metodzie.

Test będzie wyglądając w następujący sposób:

```java
@ArchTest
private static final ArchRule deprecated_methods_should_not_be_called = FreezingArchRule.freeze(methods()
	.that().areAnnotatedWith(Deprecated.class).and().haveName("hello").should(notBeCalledAtAll()));
```

Tutaj też warto zwrócić uwage, że zawięźiłem walidacje metod oznaczonych adnotacja deprecated do jedynie tych, które mają w nazwie “hello”. Nie chce ograniczać wywołań metod deprecated dla całego systemu. 

W ciągu wywołań tego DSL ArchUnit użyłem **notBeCalledAtAll**(), jest to kod który musiałem doimplementować. Jego zadaniem jest sprawdzenie czy weryfikowana metoda jest gdziekolwiek wywoływana, wykorzystałem do tego API dostarczone przez ArchUnit. 

```java
private static ArchCondition<JavaMethod> notBeCalledAtAll() {
return new ArchCondition<JavaMethod>("not be called at all") {
@Override
public void check(JavaMethod method, ConditionEvents events) {
	if (method.getCallsOfSelf().size() > 0) {
		method.getCallsOfSelf().forEach(methodCall -> {
			events.add(SimpleConditionEvent.violated(method,
					"Deprecated method is called in this locations - " + methodCall.getOriginOwner().getFullName() + " at line: " + methodCall.getLineNumber()));
		});
		return;
	}
	events.add(SimpleConditionEvent.satisfied(method, "Deprecated method is not called anywhere"));
	}
};
}
```

Teraz wystarczy przed wrzuceniem swoich zmian na repozytorium uruchomić raz test wykorzystując FreezingArchitectureRule, zarmoźić aktualne wykonania kodu, a każde nowe wywołanie będzie zgłoszone podczas uruchamiania testów jednostkowych. 

## Podsumowanie

ArchUnit jest świentym narzędziem który wpisuje się w styl pracy gdzie dostaje się szybki feedback. Informacje o popełnionych błędach otrzymujemy na etapie uruchamiania testów jednostkowych, najcześciej jeszcze na swojej maszynie przed wypchnieciem zmian na repozytorium. Biblioteka jest na tyle elastyczna, że pozwala polegać nie tylko na świetnie zbudowanym DSLu, ale także można skorzystać z API jakie dostarczane jest przez twórców i tworzyć swoje zasady, tak jak pokazałem w przypadku metody *notBeCalledAtAll.* Samo wprowadzenie biblioteki do istniejącego projektu też może okazać się nie tak trudnym zadaniem dzieki zastosowaniu ViolationStore i FreezingArchitectureRule.