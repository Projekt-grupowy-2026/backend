# ADR 0001: Stack technologiczny backendu

- **Status:** zaakceptowana
- **Data:** 2026-11-12

## Kontekst

Backend jest jedynym źródłem prawdy aplikacji logistycznej. Przechowuje plany tras (odcinki i postoje), przyjmuje pozycję GPS i prędkość z aplikacji kierowcy, aktualizuje status realizacji i przelicza ETA. Z API korzystają trzy role: dyspozytor, kierowca i klient. Klient widzi wyłącznie swój przejazd.

Repozytorium jest na początku prac i ma jeden zespół. Granice odpowiedzialności są już opisane w [`docs/README.md`](../README.md), ale nie uzasadniają osobnych deployowalnych usług.

Ta decyzja ustala język, build, framework, bazę, dane przestrzenne, tożsamość i sposób cięcia kodu na moduły. Nie ustala protokołu strumienia lokalizacji, algorytmu ETA ani kontraktu API.

## Decyzja

Backend jest modularnym monolitem.

| Element | Wybór | Wersja bazowa (2026-11-12) |
|---|---|---|
| Build | Gradle, Kotlin DSL (`build.gradle.kts`), wrapper | Gradle 9.9.1 |
| Język | Java, toolchain | 25 |
| Framework | Spring Boot | 4.1.2 (linia 4.1.x) |
| Baza aplikacji | PostgreSQL | 18 |
| Dane przestrzenne | rozszerzenie PostGIS | 3.7 |
| Sterownik JDBC | `org.postgresql:postgresql` | 42.7.14, z BOM Spring Boot |
| Mapowanie geometrii | `org.hibernate.orm:hibernate-spatial` | 7.4.13.Final, z BOM Spring Boot |
| Moduły | Spring Modulith | 2.1.2 (linia 2.1.x, zgodna z Boot 4.1) |
| Tożsamość | Keycloak, osobny proces | 26.8.2 (linia 26.8.x) |

Poprawki wewnątrz podanej linii minor nie wymagają nowego ADR. Wyjście z linii (na przykład Boot 4.2, PostgreSQL 19, Modulith 2.2, Keycloak 27) wymaga nowego ADR.

### Zależności aplikacji

Wersje Spring Boot, Hibernate i sterownika PostgreSQL biorą się z BOM-u Spring Boot. Modulith importujemy osobnym BOM-em. Oba wchodzą jako platformy Gradla. Plugin `io.spring.dependency-management` 1.1.7 kończy wsparcie na Gradle 8, więc przy Gradle 9 go nie stosujemy.

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.2"
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:4.1.2"))
    implementation(platform("org.springframework.modulith:spring-modulith-bom:2.1.2"))

    implementation("org.springframework.boot:spring-boot-starter-webmvc")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.postgresql:postgresql")
    implementation("org.hibernate.orm:hibernate-spatial")
    implementation("org.springframework.modulith:spring-modulith-starter-core")
    implementation("org.springframework.modulith:spring-modulith-starter-jpa")

    testImplementation("org.springframework.boot:spring-boot-starter-data-jpa-test")
    testImplementation("org.springframework.boot:spring-boot-starter-security-oauth2-resource-server-test")
    testImplementation("org.springframework.modulith:spring-modulith-starter-test")
}
```

Starter `spring-boot-starter-web` i `spring-boot-starter-oauth2-resource-server` są w Boot 4 oznaczone jako przestarzałe. Używamy nazw z prefiksem technologii: `webmvc` i `security-oauth2-resource-server`.

Nie używamy adaptera `keycloak-spring-boot`. Keycloak wydaje tokeny. Backend jest wyłącznie resource serverem i sprawdza JWT (issuer, audience, podpis).

### Baza i PostGIS

Jedna instancja PostgreSQL 18, dwie bazy:

- `logistics` — dane aplikacji, z włączonym `CREATE EXTENSION postgis;`
- `keycloak` — schemat Keycloaka, bez PostGIS

Obraz lokalny bazy aplikacji: `postgis/postgis:18-3.7`. Obraz Keycloaka: `quay.io/keycloak/keycloak:26.8.2`.

Układ współrzędnych to WGS 84:

- pozycja pojazdu i lokalizacja postoju: `geography(Point, 4326)`
- przebieg odcinka: `geography(LineString, 4326)`

Odległości liczymy funkcjami PostGIS (`ST_Distance` i pokrewne), w metrach. Hibernate Spatial mapuje te kolumny na typy JTS. Współrzędne nie są trzymane jako para kolumn `latitude`/`longitude` ani jako JSON.

### Keycloak

Jeden realm współdzielony przez aplikację webową i mobilną. Role realm:

| Rola | Użytkownik |
|---|---|
| `dispatcher` | dyspozytor |
| `driver` | kierowca |
| `client` | klient / odbiorca |

Hasła i sesje zostają w Keycloaku. W bazie aplikacji trzymamy tylko odwzorowanie podmiotu (`sub` z tokenu) potrzebne do przypisania trasy i do ograniczenia odczytu. Ograniczenie „klient widzi tylko swój przejazd” egzekwuje backend na zapytaniu, a nie sam fakt posiadania roli.

### Spring Modulith

Jedna aplikacja Spring Boot, cztery moduły. Moduł to bezpośredni podpakiet aplikacji. Typy w korzeniu pakietu modułu są jego API. Reszta siedzi w pakiecie `internal`. Wywołanie cudzego `internal` jest naruszeniem granicy i ma wywrócić test ArchUnit ze `spring-modulith-starter-test`.

| Moduł | Odpowiedzialność |
|---|---|
| `access` | odwzorowanie podmiotu Keycloak na uczestnika i zakres danych, jaki wolno mu czytać |
| `planning` | plany tras, odcinki, postoje, przypisanie trasy do kierowcy |
| `tracking` | pozycja GPS, prędkość, historia, zdarzenia rozpoczęcia i zakończenia trasy oraz postoju |
| `eta` | przeliczenie przewidywanej godziny przyjazdu i sygnał opóźnienia |

`tracking` publikuje zdarzenia aplikacyjne po zapisie pozycji lub zdarzenia realizacji. `eta` reaguje na nie nasłuchem. Rejestr publikacji zdarzeń Modulith (`spring-modulith-starter-jpa`) leży w PostgreSQL, więc przeliczenie ETA nie ginie razem z procesem między commitem pozycji a obsługą zdarzenia.

## Uzasadnienie

- **Gradle 9.9.1.** Linia 9.x uruchamia się na Javie 25. Gradle 8.x kompiluje ten bytecode tylko przez toolchain, ale sam demon na Javie 25 wymaga co najmniej Gradla 9.1.
- **Java 25.** Aktualne LTS, w zakresie wspieranym przez Spring Boot 4.1 (17–27).
- **Spring Boot 4.1.x.** Wspierana linia (OSS do 2027-07-31). Daje Spring Framework 7, Jakarta EE 11, Hibernate 7 i Spring Security 7, których oczekują pozostałe wybory.
- **PostgreSQL 18 + PostGIS 3.7.** Trasa ma geometrię, a pojazd ma pozycję. PostGIS liczy odległość i pozwala trzymać przebieg odcinka jako linię, bez dokładania osobnej bazy przestrzennej. PostGIS 3.7 jest pierwszą linią 3.7 z binariami pod PostgreSQL 18.
- **`org.postgresql:postgresql`.** Sterownik JDBC zarządzany przez BOM Spring Boot 4.1.2 (42.7.14). Bez niego na classpath Boot nie ma z czym rozmawiać z PostgreSQL.
- **Hibernate Spatial.** PostGIS jest rozszerzeniem serwera. Żeby kolumny `geography` były polami encji, potrzebny jest `hibernate-spatial` w tej samej wersji co Hibernate z BOM-u.
- **Keycloak.** Te same role i tokeny mają działać dla frontendu i aplikacji mobilnej. Własny serwer autoryzacji w tym repozytorium dublowałby rejestr użytkowników.
- **Spring Modulith 2.1.x.** Jedno wdrożenie, a granice z dokumentu domenowego (`planning`, `tracking`, `eta`, `access`) są sprawdzane testem, zanim ktoś je obejdzie importem. Osobne usługi nie dają tu nic, czego nie da zdarzenie aplikacyjne w jednym procesie.

## Konsekwencje

- Lokalne środowisko to dwa kontenery: PostGIS i Keycloak. Keycloak dostaje własną bazę na tej samej instancji PostgreSQL.
- Testy integracyjne bazy idą na Testcontainers z obrazem `postgis/postgis:18-3.7`, nie na H2. H2 nie ma PostGIS.
- Zmiana pozycji jest zapisem w `tracking` i zdarzeniem dla `eta`. Moduły nie wołają swoich repozytoriów nawzajem.
- Wejście do API wymaga JWT z Keycloaka. Endpointy publiczne, jeśli powstaną, muszą być wypisane wprost w konfiguracji security.
- Ta decyzja nie wybiera REST albo WebSocket dla lokalizacji, nie wybiera algorytmu ETA i nie wybiera źródła natężenia ruchu. To zostaje w sekcji „Do ustalenia” w [`docs/README.md`](../README.md).

## Odrzucone opcje

- **Maven.** Równoważny przy tym stacku. Zespół wybrał Gradle.
- **Java 21.** Nadal wspierana przez Boot 4.1, ale nie jest już aktualnym LTS.
- **Spring Boot 3.5.** Linia OSS wygasła 2026-06-30. Nowe prace zaczynamy na 4.1.
- **Baza bez PostGIS, współrzędne w dwóch kolumnach.** Wystarcza do kropki na mapie, nie wystarcza do przebiegu odcinka i odległości po kuli.
- **Adapter Keycloaka albo logowanie trzymane w aplikacji.** Adapter nie jest ścieżką wspieraną przy Spring Security 7. Trzymanie haseł w bazie aplikacji rozjeżdża się z dwoma klientami (web i mobile).
- **Mikroserwisy per moduł.** Cztery procesy, cztery wdrożenia i sieć między `tracking` a `eta` przy jednym zespole i jednym źródle prawdy.
