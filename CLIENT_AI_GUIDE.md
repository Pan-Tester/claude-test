# Instrukcja dla AI: klient GitHub Page Relay

Zaimplementuj klienta, który przesyła żądania pobrania stron przez pliki w repozytorium GitHub
i odbiera wyniki zapisane przez działający osobno serwer. Serwer wykonuje JavaScript w Chromium
i oddaje statyczną migawkę HTML oraz ZIP z assetami. Nie trzeba implementować serwera.

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

Jedynymi dozwolonymi polami są `id` i `url`. `id` można pominąć; jeśli jest podane,
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

Co około 10 sekund pobierz bieżący commit:

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

To statyczna migawka głównego dokumentu po JS. Nie odtwarza sesji, formularzy, logowania,
iframe, canvas, późniejszych API ani działania aplikacji offline. Nie usuwaj CSP z wyniku
i nie przywracaj skryptów. Linki do innych stron mogą pozostać adresami zdalnymi.

Nie zapisuj ani nie usuwaj plików w `responses/`; zarządza nimi serwer.
Nie kasuj żądania, dopóki worker pracuje. Utrzymuj mapowanie lokalnego zadania do jego ID.

## Źródła interfejsu GitHub

[GitHub Contents API — odczyt, zapis, raw i limity rozmiarów](https://docs.github.com/en/rest/repos/contents),
[GitHub References API — odczyt SHA brancha](https://docs.github.com/en/rest/git/refs#get-a-reference).
