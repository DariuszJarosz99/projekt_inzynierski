# Projekt i implementacja aplikacji webowej symulującej otwieranie paczek kart kolekcjonerskich z weryfikacją statystyczną rozkładu losowania

> Dokument przedstawia projekt główny (symulator paczek MTG) oraz, w ostatniej części, projekt alternatywny (biblioteczka książek).

## Problem i cel pracy

### Problem

Magic: The Gathering to gra karciana, w której karty kupuje się w zamkniętych paczkach. Zawartość paczki, pomimo że losowa, musi spełniać pewne reguły. Przykładowo, paczka składa się z określonej liczby miejsc (slotów), a część z nich musi zawierać kartę o konkretnej rzadkości, np. pospolitą. W paczkach istnieje także szansa na pojawienie się karty foliowanej, czyli specjalnego, błyszczącego wydania karty, które występuje z określonym prawdopodobieństwem. Reguły te różnią się w zależności od wydania gry. Aby program odwzorowywał te reguły i ich prawdopodobieństwa, potrzebny jest silnik losowania, który obsłuży wszystkie warunki. Wpisywanie ich ręcznie dla każdego wydania nie wchodzi w grę, byłoby to zbyt żmudne i podatne na błędy.

### Cel pracy

Celem pracy jest stworzenie aplikacji internetowej, w której użytkownik zakłada konto i „otwiera" wirtualne paczki z kartami, a program losuje karty tak, jak robiłaby to prawdziwa paczka. Kod aplikacji będzie zawierał ogólny mechanizm losowania, a reguły konkretnych wydań oraz dane kart będą pochodzić z zewnętrznego źródła danych (MTGJSON). Wyniki otwarć zapisują się w historii konta, a obrazy kart pochodzą z publicznej bazy Scryfall.

Praca sprawdzi metodami statystyki, czy program rzeczywiście losuje tak często, jak zakładają reguły. W tym celu program otworzy na próbę tysiące paczek dla wybranego wydania, a wyniki zostaną porównane z oczekiwaniami testem statystycznym (chi-kwadrat). Taki test pozwala stwierdzić, czy różnice między wynikiem a oczekiwaniem są zwykłym przypadkiem, czy sygnałem błędu w programie. Test wykazuje zgodność z regułami wczytanymi ze źródła danych, a nie z rzeczywistą produkcją paczek przez wydawcę gry. To ograniczenie zostanie opisane w pracy.

## Zakres projektu

Zakres podstawowy (MVP) obejmuje najważniejsze funkcjonalności, które muszą zostać wdrożone, aby program działał poprawnie:

1. Rejestracja, logowanie i wylogowanie użytkownika.
2. Lista wydań z wyborem typu paczki.
3. Otwieranie paczek, losowanie według reguł oraz wyświetlanie kart z obrazami.
4. Historia otwarć przypisana do konta, ze szczegółami każdej paczki.
5. Testy jednostkowe silnika losowania i test statystyczny rozkładu.

Deklaracja zakresu: aplikacja będzie obsługiwała kilka wybranych wydań (3 do 5) i typów paczek. Implementacja każdego wydania jest zbyt dużym zakresem dla projektu inżynierskiego.

## Problemy inżynierskie do rozwiązania

| # | Problem | Dlaczego jest trudny | Planowane rozwiązanie |
| --- | --- | --- | --- |
| 1 | Źródło reguł paczek | Scryfall podaje dane kart, ale nie opisuje składu paczek | Konfiguracja paczek i dane kart z MTGJSON; Scryfall tylko dla obrazów kart |
| 2 | Silnik losowania ważonego | Wybór szablonu paczki według wag, potem losowanie kart z arkuszy według wag, z obsługą duplikatów i foili | Czysta funkcja z wstrzykiwanym generatorem liczb losowych, więc da się ją testować deterministycznie |
| 3 | Limity Scryfalla | API Scryfall ma limity zapytań, a obrazy 10–15 kart w każdej paczce to wiele żądań | Dane kart są lokalnie (z MTGJSON), więc samo losowanie nie wysyła zapytań do Scryfall; ze Scryfall ładowane są wyłącznie obrazy, a przy większym ruchu można je zapisywać we własnym cache |
| 4 | Zgodność identyfikatorów kart | Konfiguracja paczki i obrazy kart pochodzą z dwóch różnych źródeł | Mapowanie po identyfikatorze Scryfall zapisanym przy karcie w danych MTGJSON; weryfikacja na prototypie w pierwszych dniach |
| 5 | Uczciwość losowania | Losowanie w przeglądarce można by sfałszować | Losowanie wyłącznie na serwerze, kryptograficznym generatorem liczb losowych |
| 6 | Uwierzytelnianie | Konta użytkowników wymagają bezpiecznego przechowywania haseł i obsługi sesji | Hashowanie haseł (argon2), token w ciasteczku httpOnly, walidacja danych, ograniczenie prób logowania |
| 7 | Dowód poprawności | Samo „działa" nie dowodzi zgodności z prawdopodobieństwami | Symulacja tysięcy otwarć i test zgodności rozkładu (chi-kwadrat) z oczekiwanymi częstościami |
| 8 | Struktura kodu | Logika losowania, konta i historia powinny dać się rozwijać niezależnie | Modularny monolit: jeden backend podzielony na moduły o jawnych interfejsach i jednokierunkowych zależnościach |

## Stack technologiczny

Stack składa się z aplikacji Vue w przeglądarce oraz backendu w Pythonie zbudowanego jako modularny monolit: jeden proces podzielony na trzy moduły o jasnych granicach (konta, losowanie paczek, otwarcia i historia). Wybrano go zamiast mikroserwisów świadomie. Przy jednym programiście i dwóch tygodniach pracy pozwala uniknąć narzutu komunikacji sieciowej, osobnego wdrażania i testowania wielu usług, a jednocześnie zachowuje podział, który w przyszłości pozwoli wydzielić moduły do osobnych usług.

### Frontend: Vue 3 + Vite

Frontend to część aplikacji, którą użytkownik widzi w przeglądarce: formularze logowania, lista wydań, ekran otwierania paczki i historia. Vue to framework JavaScript, w którym interfejs składa się z małych komponentów (np. pojedynczej karty), a Vite uruchamia i buduje projekt. Do podstron służy Vue Router, a do stanu aplikacji (np. kto jest zalogowany) Pinia. Wygląd zapewnia własny arkusz stylów, bez dodatkowej biblioteki komponentów. Frontend jest osobną aplikacją, która rozmawia z backendem przez JSON.

### Backend: Python + FastAPI

Backend działa na serwerze i nie jest widoczny dla użytkownika. Składa się z trzech modułów: accounts (konta, logowanie, sesje), packs (dane MTGJSON, silnik losowania i test statystyczny) oraz openings (otwieranie paczek i historia, korzysta z dwóch pozostałych). Moduły accounts i packs nie zależą od żadnych innych, a silnik losowania nie zna ani HTTP, ani bazy danych. Losowanie odbywa się wyłącznie na serwerze, żeby nie można go było sfałszować w przeglądarce. FastAPI to framework do tworzenia API, który sprawdza poprawność danych i sam generuje dokumentację.

### Baza danych: SQLite + SQLAlchemy

Baza ma trzy tabele: użytkownicy, otwarcia paczek oraz karty w każdym otwarciu. Same karty nie są w bazie, tylko ich identyfikatory z MTGJSON. SQLite to jeden plik, więc nie wymaga instalacji serwera bazy. SQLAlchemy pozwala później przejść na PostgreSQL zmianą adresu połączenia. Moduł packs nie korzysta z bazy, tylko z plików z danymi wydań.

### Logowanie: argon2 + token w ciasteczku httpOnly

Użytkownik rejestruje się loginem i hasłem. Hasła są hashowane algorytmem argon2, więc w bazie leżą tylko skróty, z których nie da się odtworzyć hasła. Po zalogowaniu aplikacja wydaje token (JWT) w ciasteczku httpOnly, niedostępnym dla JavaScriptu, co utrudnia jego kradzież. Token wygasa po 24 godzinach; jego ograniczeniem jest brak możliwości unieważnienia go przed wygaśnięciem. Dodatkowo dane z formularzy są walidowane, a liczba prób logowania ograniczona.

### Dane kart i reguł: MTGJSON + Scryfall

MTGJSON dostarcza dane kart (nazwa, rzadkość, numer) oraz reguły paczek (arkusze kart z wagami i szablony paczek); to społecznościowy zbiór danych, nie dane od wydawcy gry. Pliki wydań są pobierane z MTGJSON do lokalnego katalogu i wczytywane przy starcie aplikacji, więc losowanie nie zależy od połączenia z siecią. Scryfall służy wyłącznie do obrazów kart: adres obrazu powstaje z identyfikatora Scryfall zapisanego przy karcie w MTGJSON, a obrazy ładuje przeglądarka. Aplikacja obsługuje wydania, dla których konfiguracja paczki jest kompletna. Poza zakresem podstawowym pozostają arkusze z równoważeniem kolorów i kartami stałymi.

### Testy: pytest + SciPy

pytest sprawdza silnik losowania: m.in. czy generator z ustalonym ziarnem daje powtarzalny wynik, oraz działanie kont, limitu prób logowania i historii przez API. SciPy służy do testu chi-kwadrat: symulacja około 100 tysięcy otwarć dla wybranego wydania, a wynik porównany z oczekiwanymi częstościami trafia do pracy jako tabela. Silnik można testować bez reszty aplikacji, bo jest odseparowany od HTTP i bazy danych.

### Uruchamianie: jeden skrypt

Aplikację uruchamia jeden skrypt, który tworzy środowisko Pythona, instaluje zależności i startuje backend oraz frontend. Dane wydań pobiera się z MTGJSON jednym poleceniem. Wdrożenie w chmurze (np. w Microsoft Azure) jest możliwym rozszerzeniem projektu, ale nie jest wymagane do jego działania.

## Projekt alternatywny: aplikacja webowa do zarządzania prywatną biblioteczką książek

Tytuł roboczy: **Projekt i implementacja aplikacji webowej do zarządzania prywatną biblioteczką książek z uzupełnianiem danych bibliograficznych na podstawie numeru ISBN**

### Problem i cel pracy

#### Problem

Osoby, które dużo czytają, gromadzą książki w różnych miejscach: na domowej półce, w czytniku, u znajomych czy w bibliotece. Z czasem trudno ustalić, które pozycje już posiadają, które przeczytały, które planują przeczytać i które zostały komuś pożyczone. Najczęstsze rozwiązania, czyli notatki i arkusze kalkulacyjne, wymagają ręcznego przepisywania tytułów i autorów, nie ułatwiają wyszukiwania ani porządkowania zbioru, a korzystanie z nich na telefonie jest niewygodne. Każdą z tych czynności można zautomatyzować, jeśli dane o książce zostaną pobrane z publicznego katalogu na podstawie numeru ISBN, a cały zbiór będzie przechowywany w jednej aplikacji dostępnej z każdego urządzenia.

#### Cel pracy

Celem pracy jest zaprojektowanie i wykonanie aplikacji internetowej, w której użytkownik zakłada konto i prowadzi własną biblioteczkę. Książkę można dodać ręcznie albo wpisując numer ISBN, na podstawie którego aplikacja pobiera dane bibliograficzne (tytuł, autora, okładkę) z publicznego katalogu książek. Każdej pozycji użytkownik przypisuje status (do przeczytania, w trakcie czytania, przeczytana), ocenę i notatkę, a także może oznaczyć, że książka została pożyczona. Zbiór można wyszukiwać, filtrować i sortować, a widok statystyk podsumowuje aktywność czytelniczą. Aplikacja działa w przeglądarce, zarówno na komputerze, jak i na telefonie.

Praca obejmuje projekt architektury (modularny monolit), implementację serwera i interfejsu użytkownika oraz weryfikację poprawności rozwiązania.

#### Jak to sprawdzę

Poprawność walidacji numerów ISBN i logiki aplikacji sprawdzą testy automatyczne. Osobne testy zweryfikują izolację danych użytkowników, czyli próbę odczytu i zmiany cudzych książek, która musi zostać odrzucona. Wydajność wyszukiwania zostanie zmierzona na wygenerowanym zbiorze testowym (rzędu kilkunastu tysięcy książek) z indeksem w bazie i bez niego, a wyniki zostaną przedstawione w pracy w formie tabeli.

### Zakres projektu

Zakres podstawowy (MVP) obejmuje następujące funkcjonalności:

1. Rejestracja, logowanie i wylogowanie użytkownika.
2. Dodawanie książki ręcznie oraz po numerze ISBN, z automatycznym uzupełnieniem tytułu, autora i okładki z publicznego katalogu.
3. Lista książek z wyszukiwaniem, filtrowaniem po statusie, sortowaniem i stronicowaniem.
4. Edycja i usuwanie książek: status, ocena w skali 1–5, notatka oraz informacja o wypożyczeniu (komu i od kiedy).
5. Widok statystyk: liczba przeczytanych książek w wybranym roku i rozkład ocen.
6. Testy jednostkowe, testy API oraz pomiar wydajności wyszukiwania.

Deklaracja zakresu: poza zakresem podstawowym pozostają skanowanie kodów kreskowych kamerą, aplikacja natywna na telefon oraz współdzielenie biblioteczek między użytkownikami. Są to możliwe kierunki rozbudowy.

### Problemy inżynierskie do rozwiązania

| # | Problem | Dlaczego jest trudny | Planowane rozwiązanie |
| --- | --- | --- | --- |
| 1 | Źródło danych o książkach | Ręczne wpisywanie jest uciążliwe, a dane w różnych katalogach się różnią i bywają niekompletne | Uzupełnianie po ISBN z publicznego API katalogu książek, z możliwością ręcznej korekty i obsługą braku wyniku |
| 2 | Poprawność numerów ISBN | Numery występują w dwóch formatach (ISBN-10 i ISBN-13), a przy wpisywaniu łatwo o pomyłkę | Walidacja sumy kontrolnej, normalizacja do jednego formatu, testy jednostkowe |
| 3 | Wyszukiwanie i filtrowanie | Duży zbiór wymaga szybkiego wyszukiwania po tytule i autorze | Indeksy w bazie, stronicowanie po stronie serwera, opcjonalnie wyszukiwanie pełnotekstowe SQLite (FTS5); pomiar czasu odpowiedzi z indeksem i bez |
| 4 | Izolacja danych użytkowników | Każdy użytkownik może widzieć i zmieniać wyłącznie własne książki | Każde zapytanie filtrowane po właścicielu; testy sprawdzające brak dostępu do cudzych danych |
| 5 | Uwierzytelnianie | Konta użytkowników wymagają bezpiecznego przechowywania haseł i obsługi sesji | Hashowanie haseł (argon2), token w ciasteczku httpOnly, walidacja danych, ograniczenie prób logowania |
| 6 | Zależność od zewnętrznego API | Katalog może być niedostępny lub ograniczać liczbę zapytań | Limit czasu zapytań, obsługa błędów, dodawanie ręczne jako rozwiązanie awaryjne |
| 7 | Struktura kodu | Konta, biblioteka i klient katalogu powinny dać się rozwijać niezależnie | Modularny monolit: jeden backend podzielony na moduły o jawnych interfejsach |

### Stack technologiczny

Stack składa się z aplikacji Vue w przeglądarce oraz backendu w Pythonie zbudowanego jako modularny monolit: jeden proces podzielony na moduły o jasnych granicach. Jest to ten sam zestaw technologii, który przewidziano w projekcie głównym, więc wybór między projektami nie zmienia narzędzi.

#### Frontend: Vue 3 + Vite

Frontend to część aplikacji widoczna w przeglądarce: formularze logowania i dodawania książek, lista z wyszukiwarką i filtrami, widok szczegółów oraz statystyki. Vue to framework JavaScript, w którym interfejs składa się z małych komponentów (np. karty książki), a Vite uruchamia i buduje projekt. Do podstron służy Vue Router, a do stanu aplikacji Pinia. Interfejs jest responsywny, więc działa na telefonie, a dodanie konfiguracji PWA pozwala zainstalować go na ekranie głównym jak zwykłą aplikację.

#### Backend: Python + FastAPI

Backend składa się z trzech modułów: accounts (konta, logowanie, sesje), library (książki, statusy, oceny, wypożyczenia, statystyki) oraz catalog (klient publicznego API z danymi bibliograficznymi). Moduły komunikują się przez jawne interfejsy, a moduły accounts i catalog nie zależą od pozostałych. FastAPI to framework do tworzenia API, który sprawdza poprawność danych i sam generuje dokumentację.

#### Baza danych: SQLite + SQLAlchemy

Baza zawiera tabele użytkowników, książek (właściciel, ISBN, tytuł, autor, status, ocena, notatka) oraz wypożyczeń. Po kolumnach używanych do wyszukiwania i filtrowania zostaną założone indeksy. SQLite to jeden plik, więc nie wymaga instalacji serwera bazy, a SQLAlchemy pozwala później przejść na PostgreSQL zmianą adresu połączenia.

#### Logowanie: argon2 + token w ciasteczku httpOnly

Użytkownik rejestruje się loginem i hasłem. Hasła są hashowane algorytmem argon2, więc w bazie przechowywane są tylko skróty. Po zalogowaniu aplikacja wydaje token w ciasteczku httpOnly, niedostępnym dla JavaScriptu. Dane z formularzy są walidowane, a liczba prób logowania ograniczona.

#### Dane bibliograficzne: publiczne API katalogu książek

Dane o książce (tytuł, autor, okładka) są pobierane po numerze ISBN z publicznego katalogu; konkretne źródło zostanie wybrane po sprawdzeniu zasad korzystania i limitów zapytań. Pobrane dane trafiają do bazy użytkownika i można je poprawić ręcznie. Przy braku odpowiedzi katalogu użytkownik może wprowadzić książkę samodzielnie, więc działanie aplikacji nie zależy od zewnętrznej usługi.

#### Testy: pytest

pytest sprawdza walidację ISBN, logikę biblioteki, działanie API oraz izolację danych użytkowników. Osobny skrypt generuje zbiór testowy i mierzy czas wyszukiwania, a wyniki trafiają do pracy jako tabela.

#### Uruchamianie: jeden skrypt

Aplikację uruchamia jeden skrypt, który tworzy środowisko Pythona, instaluje zależności i startuje backend oraz frontend.
