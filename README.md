# Projekt i implementacja aplikacji webowej symulującej otwieranie paczek kart kolekcjonerskich z weryfikacją statystyczną rozkładu losowania

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
