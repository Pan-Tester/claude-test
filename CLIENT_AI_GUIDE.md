# Instrukcja dla AI: klient GitHub Page Relay

Zaimplementuj klienta, który przesyła żądania pobrania stron przez pliki w repozytorium GitHub
i odbiera wyniki zapisane przez działający osobno serwer. Serwer wykonuje JavaScript w Chromium
i oddaje statyczną migawkę HTML oraz ZIP z assetami. Nie trzeba implementować serwera.

Serwer obsługuje teraz również sesje i interakcje. Klient pokazuje aktualny widok,
wysyła akcje do Chromium na serwerze i po każdym kroku podmienia HTML/ZIP/screenshot.

## Stałe i dostęp

- Repozytorium: `Pan-Tester/claude-test`.
- Branch: `main`.
- Baza API: `https://api.github.com/repos/Pan-Tester/claude-test`.
- Token pobieraj ze zmiennej środowiskowej `GITHUB_TOKEN` lub konfiguracji użytkownika.
- Zapis żądań wymaga repository Contents: write. Odczyt wymaga Contents: read;
  obecnie repo jest publiczne, więc sam odczyt może działać bez tokena.
- Nie zapisuj tokena w plikach JSON, kodzie źródłowym, URL-ach ani logach.
- Serwer musi być uruchomiony, żeby nowe żądania dostały odpowiedź. Sam GitHub nie przetwarza żądań.

Nagłówki żądań do GitHub API:

```http
Authorization: Bearer <GITHUB_TOKEN>
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2026-03-10
User-Agent: GitHubPageRelayClient/1.0
```

## 1. Przygotuj żądanie

Wygeneruj unikalne ID, np. `req-` + UUID v4. ID ma 1–80 znaków i pasuje do
`^[A-Za-z0-9][A-Za-z0-9_-]{0,79}$`.

Utwórz bajty UTF-8 pliku `requests/<id>.json`, np.:

```json
{
  "id": "req-12345678-1234-4234-8234-123456789abc",
  "url": "https://www.google.com/"
}
```

Dozwolone pola to `id`, `url`, `session_id`, `expected_revision`, `actions` i `close_session`.
`id` można pominąć; jeśli jest podane,
musi zgadzać się z nazwą pliku bez `.json`. Maksymalny rozmiar żądania to 16 KiB.
Adresy HTTP/HTTPS są obsługiwane na portach 80/443; można też podać `google.com`.

Nie dodawaj pól z nagłówkami, cookies, opisem zadania, konfiguracją przeglądarki ani tokenem.
Nie używaj ponownie ID zakończonego zadania, nawet jeśli jego wynik ma status `error`.

## 2. Wgraj plik na GitHub

Zakoduj **cały plik JSON jako bajty UTF-8 w Base64**. Base64 to kodowanie transportowe,
nie dodatkowe pole wewnątrz żądania relay.

```http
PUT /repos/Pan-Tester/claude-test/contents/requests/<id>.json
```

Body GitHub API:

```json
{
  "message": "relay: request <id>",
  "branch": "main",
  "content": "<BASE64_BAJTÓW_PLIKU_JSON>"
}
```

Przy tworzeniu nowego pliku **nie podawaj `sha`**. Oczekuj HTTP 201. Zapisz ID,
`content.sha` oraz `commit.sha` zwrócone przez GitHub. `content.sha` identyfikuje żądanie
i może być porównane z `request_sha` odpowiedzi serwera; to hash Git, nie hash SHA-256 ZIP-a.

Przy utracie odpowiedzi HTTP sprawdź, czy istnieje żądanie lub wynik pod tym samym ID,
zanim ponowisz wysyłanie. Worker mógł już przetworzyć i usunąć plik wejściowy.
Przy konflikcie z innym commitem ponów zapis tego samego, wcześniej zweryfikowanego ID
po krótkim opóźnieniu. Nie nadpisuj cudzych plików. Wysyłaj kolejne pliki sekwencyjnie.

## 3. Czekaj na wynik

Przed pierwszym pollowaniem sprawdź dostęp do repo/brancha. Nie traktuj nieistniejącego
repo albo braku dostępu jako oczekiwania na wynik.

Co około 3–5 sekund pobierz bieżący commit:

```http
GET /repos/Pan-Tester/claude-test/git/ref/heads/main
```

Zapisz `object.sha` jako `COMMIT_SHA`. Następnie odczytaj odpowiedź z tego commita:

```http
GET /repos/Pan-Tester/claude-test/contents/responses/<id>.json?ref=<COMMIT_SHA>
Accept: application/vnd.github.raw+json
```

Ten wariant `Accept` zwraca bezpośrednio bajty pliku, także dla JSON-ów większych niż 1 MiB.
Nie dekoduj ich ponownie z Base64. Przy HTTP 200 zdekoduj UTF-8 i parsuj JSON.
HTTP 404, po potwierdzeniu dostępu do repo i brancha, oznacza brak wyniku w tym commicie.
Powtórz odczyt z nowym SHA brancha. Ustaw konfigurowalny timeout klienta, np. 300 sekund.
Timeout klienta nie anuluje zadania na serwerze; zachowaj ID do późniejszego odbioru.

Nie uznawaj zniknięcia `requests/<id>.json` za błąd — worker usuwa wejście przy publikacji.
Nie pobieraj dużych plików przez zwykłe pole `content` z Contents API: dla 1–100 MiB
nie jest ono dostępne w standardowy sposób. Użyj nagłówka raw pokazanej wyżej metody.

HTTP 401/403 może oznaczać problem z tokenem, uprawnieniami albo limit API.
Odczytaj nagłówki `Retry-After`, `X-RateLimit-Remaining` i `X-RateLimit-Reset`;
dla limitu odczekaj wskazany czas, dla błędu uprawnień przerwij z czytelnym komunikatem.
Błędy 5xx i przejściowe błędy transportu ponawiaj z rosnącym opóźnieniem.

## 4. Zinterpretuj odpowiedź

Wymagaj `schema_version: 1` i `id` zgodnego z wysłanym żądaniem.
Jeśli zachowano SHA wejścia, porównaj `request_sha` z otrzymanym wcześniej `content.sha`.

Sukces:

```json
{
  "schema_version": 1,
  "id": "<id>",
  "status": "done",
  "request_sha": "<GIT_BLOB_SHA_ŻĄDANIA>",
  "requested_url": "https://www.google.com/",
  "final_url": "https://www.google.com/",
  "title": "Google",
  "created_at": "<ISO_8601_UTC>",
  "html": "<HTML_MIGAWKI>",
  "assets": [
    {
      "url": "https://www.google.com/logo.png",
      "path": "assets/<hash>.png",
      "content_type": "image/png",
      "bytes": 1234,
      "sha256": "<SHA256_PLIKU_W_ZIP>"
    }
  ],
  "warnings": [],
  "zip": {
    "path": "responses/<id>.zip",
    "bytes": 5678,
    "sha256": "<SHA256_ZIP>"
  },
  "snapshot": {
    "javascript_enabled_during_capture": true,
    "javascript_removed_from_output": true
  }
}
```

Skrócone przykłady w tej instrukcji zawierają placeholdery. Faktyczne pola liczbowe,
hashe, tytuł i URL po przekierowaniach pochodzą z odpowiedzi serwera.
`warnings` może być niepuste przy `status: done`: pokaż, że migawka może być niepełna.

Błąd:

```json
{
  "schema_version": 1,
  "id": "<id>",
  "status": "error",
  "request_sha": "<GIT_BLOB_SHA_ŻĄDANIA>",
  "created_at": "<ISO_8601_UTC>",
  "error": {
    "type": "ValueError",
    "message": "Page returned HTTP 404"
  }
}
```

Przy `error` pokaż komunikat i nie oczekuj ZIP-a. Ponowienie to nowe ID i nowy plik wejściowy.

## 5. Pobierz ZIP i wyświetl migawkę

Dla `done` pobierz ZIP **z tego samego `COMMIT_SHA`, z którego odczytano JSON**:

```http
GET /repos/Pan-Tester/claude-test/contents/responses/<id>.zip?ref=<COMMIT_SHA>
Accept: application/vnd.github.raw+json
```

Zapisz odpowiedź jako bajty, bez parsowania JSON i bez dekodowania Base64.
Weryfikuj długość względem `zip.bytes` i SHA-256 bajtów względem `zip.sha256`.
JSON i ZIP są publikowane razem w jednym commicie. Brak ZIP-a przy `done` w tym samym
commicie lub błędny hash oznacza nieprawidłowy wynik; nie pokazuj go jako kompletnego.

Rozpakuj ZIP do osobnego katalogu dla tego ID, zachowując ścieżki:

```text
index.html
manifest.json
assets/...
```

Nie dopuszczaj przy rozpakowaniu ścieżek absolutnych ani wyjścia przez `..` poza katalog.
Opcjonalnie sprawdź SHA-256 każdego assetu z `assets`/`manifest.json`.
Wyświetl `index.html` przez lokalny serwer HTTP lub odpowiedni sandboxowany podgląd
obsługujący względne ścieżki. Samo wklejenie `html` do dokumentu pod obcym URL-em
nie zapewni poprawnego ładowania `assets/...`.

To statyczna migawka głównego dokumentu po JS. Interakcje, formularze i kolejne API wykonują
się w sesji na serwerze, a klient odbiera nowy widok. Nie uruchamiaj kopii jako samodzielnej
aplikacji offline. Nie usuwaj CSP i nie przywracaj skryptów. Linki do innych stron mogą
pozostać adresami zdalnymi — klient powinien przechwytywać kliknięcia. Canvas obejrzyj
na `screenshot.png`; kliknięcia współrzędnymi dotyczą widoku na tym screenshocie.

Nie zapisuj ani nie usuwaj plików w `responses/`; zarządza nimi serwer.
Nie kasuj żądania, dopóki worker pracuje. Utrzymuj mapowanie lokalnego zadania do jego ID.

## 6. Stała sesja i akcje — obowiązujący protokół interakcji

Otwórz sesję, wybierając unikalne `session_id` z takimi samymi ograniczeniami jak ID żądania:

```json
{
  "id":"req-open-001",
  "session_id":"session-browser-abc",
  "url":"https://www.google.com/"
}
```

Odpowiedź `done` zachowuje dotychczasowe pola oraz dodaje:

```json
{
  "session_id":"session-browser-abc",
  "revision":1,
  "session_ttl_seconds":900,
  "actions_completed":0,
  "elements":[
    {
      "element_id":"e12",
      "selector":"[data-relay-id=\"e12\"]",
      "tag":"textarea",
      "type":null,
      "text":"Search",
      "href":null,
      "disabled":false,
      "visible":true,
      "rect":{"x":50,"y":100,"width":400,"height":40}
    }
  ],
  "viewport":{"width":1365,"height":768,"scroll_x":0,"scroll_y":0},
  "screenshot":{"path":"screenshot.png","bytes":12345,"sha256":"<SHA256_PNG>"}
}
```

To uzupełnienie pełnej odpowiedzi, nie osobny plik. `elements` opisuje do 1000 elementów
głównego dokumentu. Te same ID są w HTML jako `data-relay-id`. Mogą zmienić się po nawigacji,
więc zawsze korzystaj z najnowszego widoku i najnowszej wersji. `screenshot.png` jest w ZIP-ie.

Kontynuuj z nowym ID, **bez `url`**, jeśli strona ma pozostać otwarta:

```json
{
  "id":"req-action-002",
  "session_id":"session-browser-abc",
  "expected_revision":1,
  "actions":[
    {"type":"fill","element_id":"e12","value":"test relay"},
    {"type":"press","element_id":"e12","key":"Enter"}
  ]
}
```

Podany `e12` jest przykładem: odczytaj faktyczne ID pola z odpowiedzi. Serwer utrzymuje
ten sam kontekst Chromium, DOM, cookies, storage i historię. HTTP formularzy i API jest
obsługiwane, także POST/PUT/PATCH/DELETE. Token GitHuba nie jest przekazywany do stron.

`expected_revision` musi odpowiadać ostatniej odpowiedzi dla tej sesji. Jest wymagane
dla każdej zmiany istniejącej sesji, również nawigacji i zamknięcia. Pierwsze otwarcie
może je pominąć lub użyć `0`. Przed wysłaniem kolejnego kroku poczekaj na wynik poprzedniego.
Każdy wykonany krok zwiększa wersję. Stara wersja daje `status: error` i aktualną `revision`,
bez wykonania akcji. Nie odtwarzaj automatycznie starego kliknięcia na nowym widoku.

Obsługiwane akcje:

| type | Pola | Efekt |
| --- | --- | --- |
| `click` | `selector` albo `element_id`; alternatywnie `x` i `y` | Kliknięcie elementu lub punktu viewportu |
| `fill` | cel + `value` jako string | Zastąpienie tekstu w polu |
| `press` | cel + `key`, np. `Enter`, `Control+A` | Klawisz w danym elemencie |
| `select` | cel + `value` jako string | Wybór opcji select |
| `check` | cel + `checked` jako boolean | Ustawienie checkboxa/radio |
| `hover` | cel | Najechanie kursorem |
| `scroll` | `x`, `y` jako liczby całkowite; brak osi = 0 | Przewinięcie o podaną liczbę pikseli |
| `wait` | `ms` od 0 do 10000 | Oczekiwanie |
| `wait_for` | cel + opcjonalne `state` | Oczekiwanie na visible/hidden/attached/detached; domyślnie visible |
| `navigate` | `url` | Otwarcie nowego adresu w tej samej sesji |
| `back`, `forward`, `reload` | brak dodatkowych pól | Historia lub odświeżenie strony |

Cel oznacza dokładnie jedno z pól `selector` lub `element_id`. Selector powinien wskazywać
jeden element. Dla kliknięcia współrzędnymi nie podawaj celu. Współrzędne kliknięcia są
nieujemne, dotyczą pikseli CSS aktualnego viewportu, a nie współrzędnych całej strony.
Zakres osi to -20000..20000; kliknięcie nie dopuszcza wartości ujemnych. Limity: 20 akcji,
16 KiB całego JSON-a, 8 sekund na akcję i 60 sekund na cały krok domyślnie.
Nie ma akcji wykonującej dowolny kod JavaScript.

W interfejsie klienta przechwytuj kliknięcia i formularze. Znajdź najbliższy element
z `data-relay-id`, zatrzymaj lokalną nawigację i wyślij akcję do serwera. Lokalnie edytowane
wartości pól prześlij jako `fill` przed kliknięciem przycisku lub `press: Enter`. Podczas
oczekiwania blokuj kolejne zmiany w tej samej sesji. Po odebraniu wyniku podmień katalog
assetów i widok, zachowaj nowe `revision` oraz listę `elements`. Dla screenshotu uwzględnij
skalowanie obrazu przy przeliczaniu współrzędnych. Nie włączaj skryptów z pobranej strony.

Odświeżenie widoku bez zmiany adresu:

```json
{"id":"req-refresh-003","session_id":"session-browser-abc","expected_revision":2}
```

Zamknięcie sesji:

```json
{"id":"req-close-004","session_id":"session-browser-abc","expected_revision":3,"close_session":true}
```

Odpowiedź zamknięcia ma `status: closed`, `session_id` i `revision`, **bez ZIP-a i HTML-a**.
Uwzględnij ten trzeci status w parserze klienta. `close_session` nie może mieć `url` ani akcji.

Sesje wygasają po 900 sekundach bezczynności; serwer utrzymuje maksymalnie 10 sesji.
Restart usuwa konteksty. `session_expired: true` w błędzie oznacza konieczność nowego
otwarcia z nowym `session_id`. Nie odtwarzaj automatycznie wysłanych formularzy.
JavaScript jest wstrzymany między krokami; `wait` uruchamia go na czas oczekiwania.

Błąd interakcji może zawierać `actions_completed`, `session_id` i zwiększoną `revision`.
Wcześniejsze akcje z batcha mogły się udać; nie cofają się automatycznie. Pobierz widok
żądaniem bez akcji i zdecyduj o kolejnym kroku. Gotowe wyniki są buforowane przed publikacją,
żeby ponowienie GitHub API nie wykonywało znów akcji. Nagła awaria podczas samej akcji
nie gwarantuje wycofania zmian na zewnętrznej stronie.

WebSocket/EventSource/Service Worker są wyłączone. Dialogi JS są odrzucane automatycznie.
Nowa karta po kliknięciu staje się stroną sesji, a poprzednia karta jest zamykana.
Przekierowanie dokumentu 307/308 z body nie jest obsługiwane. Repo jest publiczne:
nie przesyłaj haseł ani innych poufnych wartości w JSON-ach; są zapisane w historii Git.

## Źródła interfejsu GitHub

[GitHub Contents API — odczyt, zapis, raw i limity rozmiarów](https://docs.github.com/en/rest/repos/contents),
[GitHub References API — odczyt SHA brancha](https://docs.github.com/en/rest/git/refs#get-a-reference).
