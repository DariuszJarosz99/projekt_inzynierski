# Projekt i implementacja aplikacji webowej symulującej otwieranie kart kolekcjonerskich z weryfikacją statystyczną rozkładu losowania
## Problem i cel pracy
### Problem
Magic: The Gathering to gra karciana, w której karty kupuje się w zamkniętych paczkach. Zawartość paczki pomimo że losowa, musi spełniać pewne reguły. Przykładowo, jedna paczka ma w środku 15 slotów z kartami. Pierwsze 6 slotów zawsze muszą zawierać kartę z rzadkością "pospolita". W paczkach istnieje szansa na pojawienie się kart foliowanych, czyli specjalnego wydania karty która jest niezwykle rzadka. Aby mieć pewność że program wiernie odwzorował rzeczywiste szanse, wymaganym jest stworzenie silnika reguł losowania który będzie w stanie obsłużyć wszystkie warunki. Wpisywanie wszystkiego ręcznie nie wchodzi w grę, byłoby to zbyt żmudne i podatne na błędy.
### Cel pracy
Celem pracy jest stworzenie aplikacji internetowej, w której uzytkownik zakłada konto i "otwiera" wirtualne paczki z kartami, a program losuje kart tak, jak robiłaby to prawdziwa paczka. Kod aplikacji będzie zawierał ogólny mechanizm losowani, reguły konkretnych wydań będą jednak pochodzić z zewnętrznego źródła. Praca będzie sprawdzała metodami statystyki, czy program rzeczywiście losuje tak często, jak zakładają reguły. Wyniki otwarć zapisują się w historii konta, a dane i obrazy kart pochodzą z publicznej bazy "Scryfall". Aby to sprawdzić, program otworzy na próbę tysiące paczek dla wybranego wydania, a wyniki zostaną porównane z oczekiwaniami testem statystycznym (chi-kwadrat). Taki test powie mi czy różnice między wynikiem a oczekiwaniem są zwykłym przypadkiem, czy sygnałem błędu w programie.

## Zakres projektu
Wersja bazowa projektu ma obejmować najważniejsze funkcjonalności które muszą zostać wdrożone aby program działał poprawnie

1. Rejestracja, logowanie i wylogowanie użytkownika
2. Lista wydań z wyborem typu paczki
3. Otwieranie paczek, losowanie według reguł oraz wyświetlanie kart z obrazami
4. Historia otwarć przypisana do konta, ze szczegółami każdej paczki.
5. Testy jednostkowe silnika losowania i test statystyczny rozkładu.

Deklaracja zakresu: Aplikacja będzie korzystała z kilku wybranych wydań i typów paczek. Implementacja każdego wydania jest zbyt dużym zakresem dla projektu inżynierskiego

## Problemy inżynierskie do rozwiązania
| # | Problem | Dlaczego jest trudny | Planowane rozwiązanie |
| -------- | ------- | ------- | ------- |
| 1 | Źródło reguł paczek | Scryfall podaje dane kart, ale nie składa paczek | Konfiguracja paczek z MTGJSON, Scryfall tylko dla danych i obrazów kart |
| 2 | Silnik losowania ważonego | Wybór szablonu paczki według wag, potem losowanie kart z arkuszy według wag, z obsługą duplikatów i foili | Czysta funkcja z wstrzykiwanym generatorem liczb losowych, więc da się ją testować deterministycznie |
| 3 | Limity Scryfalla | Wyszukiwanie ma limit 2 zapytania na sekundę | Lokalny cache danych kart wypełniany hurtowo i odświeżany okresowo; obrazy ze Scryfall CDN, które nie ma limitów |
| 4 | Zgodność identyfikatorów kart | Konfiguracja paczki i dane kart pochodzą z dwóch różnych źródeł | Mapowanie po identyfikatorze Scryfall zapisanym przy karcie w danych MTGJSON; weryfikacja na prototypie w pierwszych dniach |
| 5 | Uczciwość losowania | Losowanie w przeglądarce można by sfałszować | Losowanie wyłącznie na serwerze, kryptograficznym generatorem liczb losowych |
| 6 | Uwierzytelnianie | Bezpieczeństwo haseł i sesji jest kluczowym problemem każdego projektu | Hashowanie haseł (argon2 lub bcrypt), token w ciasteczku httpOnly, walidacja danych, ograniczenie prób logowania |
| 7 | Dowód poprawności | Samo „działa” nie dowodzi zgodności z prawdopodobieństwami | Symulacja  tysięcy otwarć i test zgodności rozkładu (chi-kwadrat) z oczekiwanymi częstościami |

