# Backend – rola w projekcie

## 1. Kontekst

Projekt to aplikacja logistyczna wspomagająca planowanie i realizację transportu. System monitoruje lokalizację pojazdów i na bieżąco aktualizuje przewidywany czas dotarcia (ETA) do kolejnych punktów trasy.

System składa się z trzech elementów:

| Element | Repozytorium | Użytkownicy |
|---|---|---|
| Back-end / serwer | **backend** (to repo) | – |
| Aplikacja webowa | [frontend](https://github.com/Projekt-grupowy-2026/frontend) | dyspozytor, klient |
| Aplikacja mobilna | [mobile-app](https://github.com/Projekt-grupowy-2026/mobile-app) | kierowca |

Backend jest jedynym źródłem prawdy o stanie systemu – obie aplikacje klienckie komunikują się wyłącznie z nim.

## 2. Odpowiedzialności

### 2.1. Przechowywanie danych

- Plany tras – uporządkowana sekwencja **odcinków** i **postojów**.
- Odcinek: punkt początkowy i końcowy, przebieg na mapie, dystans [km], planowany czas przejazdu, przewidywana godzina przyjazdu, status realizacji.
- Postój: lokalizacja, planowana godzina rozpoczęcia, planowany czas trwania, rodzaj (rozładunek, załadunek, przerwa kierowcy, przystanek), status realizacji.
- Użytkownicy i role: dyspozytor, kierowca, klient/odbiorca.
- Przypisania tras do kierowców.
- Historia lokalizacji i prędkości pojazdów.

### 2.2. Odbieranie danych z urządzeń mobilnych

- Przyjmowanie lokalizacji GPS oraz aktualnej prędkości z aplikacji kierowcy.
- Przyjmowanie zdarzeń: rozpoczęcie/zakończenie trasy, rozpoczęcie/zakończenie postoju, potwierdzenie wykonania etapu.

### 2.3. Aktualizacja statusów

- Wyznaczanie aktualnego etapu przejazdu (który odcinek / postój jest w trakcie).
- Zmiana statusów odcinków i postojów na podstawie zdarzeń od kierowcy i pozycji pojazdu.

### 2.4. Obliczanie ETA

- Wyliczanie przewidywanej godziny przyjazdu do każdego kolejnego punktu trasy.
- Ponowne przeliczanie ETA przy każdej aktualizacji pozycji / prędkości.
- Wykrywanie opóźnień względem planu i udostępnianie tej informacji dyspozytorom i klientom.

### 2.5. Udostępnianie danych (API)

Dla **dyspozytora** (via frontend):

- CRUD planów tras, odcinków i postojów.
- Przydzielanie tras kierowcom.
- Lista pojazdów z aktualną pozycją (mapa).
- Statusy przejazdów, ETA i opóźnienia.

Dla **kierowcy** (via mobile-app):

- Pobranie przydzielonego planu trasy.
- Wysyłanie lokalizacji i prędkości.
- Zgłaszanie zdarzeń realizacji (start/koniec trasy, start/koniec postoju).

Dla **klienta / odbiorcy** (via frontend):

- Status jego dostawy.
- Przewidywany czas przyjazdu.
- Informacja o opóźnieniu.

### 2.6. Bezpieczeństwo i dostęp

- Uwierzytelnianie użytkowników i rozróżnianie ról.
- Klient ma dostęp wyłącznie do informacji dotyczących jego przejazdu.
- Podgląd pozycji kierowcy dostępny tylko dla uprawnionych użytkowników.

## 3. Scenariusze, które backend musi obsłużyć

### A. Transport ciężarowy

Kontener z portu (A) do dwóch magazynów (B, C) i powrót do A. Postoje w B (30 min, rozładunek) i C (20 min, rozładunek). W trakcie przejazdu backend aktualizuje ETA do magazynów; pracownicy magazynów sprawdzają je przez aplikację webową.

### B. Transport autobusowy

Kurs z sekwencją przystanków i godzin odjazdów. Backend monitoruje pozycję autobusu i przy opóźnieniu przelicza ETA dla kolejnych przystanków; dyspozytor śledzi realizację kursu.

## 4. Poza zakresem tego repozytorium

- Interfejs użytkownika (webowy → `frontend`, mobilny → `mobile-app`).
- Pobieranie danych z sensorów telefonu (GPS) – odpowiada za to `mobile-app`.

## 5. Do ustalenia

- Stack technologiczny — ustalony w [ADR 0001](adr/0001-stack-technologiczny.md).
- Protokół przesyłania lokalizacji (REST / WebSocket / inne).
- Algorytm obliczania ETA i źródło danych o trasach / ruchu drogowym.
- Format i wersjonowanie API.
