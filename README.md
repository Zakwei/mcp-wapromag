# mcp-wapromag

Serwer [MCP](https://modelcontextprotocol.io) (stdio) dla **WAPRO Mag** na SQL Server. Pozwala asystentowi AI (Claude Code, Claude Desktop i innym klientom MCP) czytać dane z WAPRO Mag. Na bazie testowej może też wywoływać endpointy API WAPRO (`Api_<op>Pre`, ok. 1400 operacji: faktury, dokumenty magazynowe, zamówienia, kontrahenci, kasa…).

> **Oprogramowanie płatne.** Serwer uruchamia się tylko z ważnym kluczem licencji (`WAPRO_LICENSE_KEY`), który kupujesz u właściciela repozytorium. Warunki: [LICENSE](LICENSE).
>
> Projekt niezależny, niezwiązany z producentem WAPRO. WAPRO jest znakiem towarowym jego właściciela. Paczka nie zawiera żadnych plików ani danych WAPRO: katalog endpointów generujesz z własnej, licencjonowanej instalacji (sekcja [Katalog endpointów](#3-katalog-endpointów)). Testowane na WAPRO Mag 9.20 i SQL Server 2022.

## Dwa tryby

| | **Tylko do odczytu** (produkcja) | **Pełny** (wyłącznie baza testowa) |
|---|---|---|
| Kiedy | brak `WAPRO_SQL_USER`/`WAPRO_SQL_PASSWORD` | z loginem zapisu |
| Login SQL | tylko `mcp_wapro_ro` (SELECT, bez EXECUTE) | `mcp_wapro_ro` + `mcp_wapro` |
| Procedury WAPRO | nigdy nie są uruchamiane | przez `wapro_read` / `wapro_write` |
| Narzędzia | 4 | 6 |
| Zapis | niemożliwy z samych uprawnień, niezależnie od kodu | dry-run (ROLLBACK) albo commit z `WAPRO_ALLOW_WRITE=1` |

Tryb pełny działa tylko na kopii bazy. Część procedur WAPRO przy błędzie sama cofa transakcję, a to, co wykonają potem, zapisuje się nawet w dry-run (sekcja [Dry-run bywa niepełny](#dry-run-bywa-niepełny)).

## Narzędzia

| Narzędzie | Tryb | Do czego |
|---|---|---|
| `wapro_find_endpoints` | oba | szuka endpointów po nazwie, opisie i obszarze (`query`, `area?`, `kind?`, `limit?`) |
| `wapro_describe_endpoint` | oba | pola `data`/`filter`, kody błędów z treścią, widoki `Api_V_*`, czynności, `warnings`, `policy`; `includeSource` dodaje T-SQL (tylko tryb pełny) |
| `wapro_activity` | oba | opis czynności biznesowej jako sekwencja endpointów (jeśli masz opisy czynności, sekcja [Opisy czynności](#opisy-czynności-opcjonalnie)) |
| `wapro_sql` | oba | SELECT przez login tylko do odczytu, sekrety zamaskowane (szczegóły niżej) |
| `wapro_read` | pełny | endpointy `kind = read` (`filter`, `take`, `skip`), zawsze ROLLBACK |
| `wapro_write` | pełny | dowolny endpoint: `dryRun: true` (ROLLBACK) albo commit |

`wapro_sql`:
- `maxRows` ≤ 5000;
- osobne połączenie zamykane po każdym wywołaniu, transakcja zawsze cofana;
- zapytanie musi zaczynać się od `SELECT`, `WITH` lub `DECLARE`;
- odrzuca `EXEC`, polecenia transakcji, `WAITFOR`, `OPENQUERY`/`OPENROWSET` i wskazówki blokad (`TABLOCK`, `UPDLOCK`, `HOLDLOCK`, `SERIALIZABLE`…), także gdy te słowa stoją wewnątrz tekstu.

## Instalacja

### Wymagania

- WAPRO Mag na SQL Server, uprawnienia sysadmina na czas zakładania loginów (raz).
- Maszyna z serwerem MCP (Windows). Nie wymaga instalowania Node.js — środowisko jest wbudowane w plik `.exe`.
- **Python 3** tylko raz, do wygenerowania katalogu z Twojej bazy (krok 3).
- Dostęp sieciowy z tej maszyny do SQL Server (TCP 1433), z logowaniem SQL (tryb mieszany).
- Klient MCP, np. [Claude Code](https://docs.claude.com/en/docs/claude-code) albo Claude Desktop.

### 0. Klucz licencji

**Cena: 2 900 zł netto jednorazowo** za licencję wieczystą na jedną firmę (NIP), z 12 miesiącami aktualizacji. **Przedłużenie aktualizacji: 580 zł netto za kolejny rok.** Po końcu okresu aktualizacji posiadana wersja działa dalej bez ograniczeń; nowsze wersje wymagają przedłużenia.

Kup klucz u właściciela repozytorium (kontakt przez profil GitHub), podając NIP firmy z WAPRO. Klucz to jeden ciąg znaków, ważny dla wskazanego odbiorcy i okresu, przypisany do tego NIP: serwer startuje tylko wtedy, gdy w podłączonej bazie WAPRO jest firma z tym NIP. Serwer sprawdza go przy starcie, bez połączenia z internetem. Bez ważnego klucza kończy pracę z komunikatem „Brak klucza licencji”, „Nieprawidłowy klucz licencji”, „Licencja wygasła” albo „Licencja obejmuje wersje wydane do …” (wersja nowsza niż okres aktualizacji: użyj wcześniejszej wersji z zakładki Releases albo przedłuż aktualizacje).

### 1. Pobranie

Z zakładki [Releases](https://github.com/Zakwei/mcp-wapromag/releases) pobierz paczkę **`mcp-wapromag-win-x64-<wersja>.zip`** (Windows) i rozpakuj, np. do `C:\mcp-wapromag`. Dalsze polecenia uruchamiasz w tym folderze.

Paczka to jeden plik `mcp-wapromag.exe` z wbudowanym Node.js i kodem w postaci bajtowej (nie trzeba instalować Node), a obok: `SZYBKI-START.md` (5 kroków), skrypty SQL (`scripts/`), narzędzia do katalogu (`tools/recon/`) i licencje bibliotek open source (`THIRD-PARTY-NOTICES.txt`). Nie zawiera kodu źródłowego.

### 2. Login SQL tylko do odczytu

Na serwerze SQL (wiersz poleceń z uprawnieniami sysadmina), dla bazy WAPRO:

```
sqlcmd -S localhost -E -b -i scripts\setup-logins.sql -v DB=<nazwa_bazy_WAPRO> RO_PASS=<silne_hasło>
```

Skrypt jest idempotentny i tworzy login `mcp_wapro_ro`:
- `db_datareader`;
- `DENY EXECUTE` na schemacie `dbo`;
- `DENY SELECT` na kolumnach z hasłami i tokenami.

Login jest serwerowy, wspólny dla wszystkich baz. Ponowne uruchomienie ustawia hasło od nowa, więc podawaj za każdym razem to samo.

### 3. Katalog endpointów

Serwer potrzebuje pliku `docs/wapro-api/catalog.json` z opisem endpointów Twojej wersji WAPRO. Generujesz go z własnej bazy. Bez niego serwer nie wystartuje i poda komunikat z odesłaniem do tej sekcji.

1. Skopiuj `tools/recon/dump_schema.ps1` i `tools/recon/dump_dictionaries.ps1` na serwer SQL i uruchom w PowerShell (uwierzytelnianie Windows, konto z dostępem do bazy):
   ```
   .\dump_schema.ps1 -Database <nazwa_bazy_WAPRO>
   .\dump_dictionaries.ps1 -Database <nazwa_bazy_WAPRO>
   ```
   - Wynik trafia do `C:\WaproMcpDump\` i `C:\WaproMcpRef\`.
   - Opcjonalnie `-Server <instancja>`, domyślnie `localhost`.
   - Moduły `WITH ENCRYPTION` zostają zaszyfrowane, serwer ich nie potrzebuje.
2. Przenieś oba katalogi na maszynę z aplikacją, np. do `/tmp/WaproMcpDump` i `/tmp/WaproMcpRef` (albo zostaw na tym samym komputerze).
3. W folderze aplikacji:
   ```
   mkdir -p /tmp/wapro-cat docs/wapro-api
   python3 -I tools/recon/extract_catalog.py /tmp/WaproMcpDump /tmp/wapro-cat
   python3 -I tools/recon/gen_docs.py /tmp/wapro-cat/catalog.json /tmp/WaproMcpRef docs/wapro-api
   ```
   Na Windows (wszystko na jednym komputerze):
   ```
   mkdir C:\wapro-cat
   py -I tools\recon\extract_catalog.py C:\WaproMcpDump C:\wapro-cat
   py -I tools\recon\gen_docs.py C:\wapro-cat\catalog.json C:\WaproMcpRef docs\wapro-api
   ```
   `gen_docs.py` zapisuje `docs/wapro-api/catalog.json` i czytelną dokumentację `docs/wapro-api/*.md`.
4. Usuń katalogi tymczasowe (`C:\WaproMcpDump`, `C:\WaproMcpRef`, `C:\wapro-cat`, `/tmp/WaproMcp*`, `/tmp/wapro-cat`). Zawierają pełne definicje procedur.

Po aktualizacji WAPRO powtórz kroki 1–4. Do tego czasu serwer w trybie pełnym widzi nowe endpointy przez katalog na żywo, ale bez opisu pól i bez klasyfikacji bezpieczeństwa: polityka traktuje je jak zwykły zapis, a wynik ma ostrzeżenie.

### 4. Konfiguracja

Zmienne środowiskowe (przekazuje je klient MCP, krok 5):

| Zmienna | Wymagana | Opis |
|---|---|---|
| `WAPRO_LICENSE_KEY` | tak | klucz licencji z kroku 0 |
| `WAPRO_SQL_SERVER` | tak | host SQL Server |
| `WAPRO_SQL_PORT` | nie | domyślnie `1433` |
| `WAPRO_SQL_DATABASE` | tak | nazwa bazy WAPRO |
| `WAPRO_SQL_RO_USER`, `WAPRO_SQL_RO_PASSWORD` | tak | login z kroku 2 (`mcp_wapro_ro`) |
| `WAPRO_SQL_USER`, `WAPRO_SQL_PASSWORD` | tylko tryb pełny | login zapisu `mcp_wapro`; oba albo żaden |
| `WAPRO_USER_ID`, `WAPRO_COMPANY_ID` | tylko tryb pełny | użytkownik API WAPRO (wiersz w `WAPROERP_MANDANT_USER`) i domyślna firma |
| `WAPRO_WAREHOUSE_ID`, `WAPRO_CASH_ID` | nie | domyślny magazyn i kasa (tryb pełny) |
| `WAPRO_ALLOW_WRITE`, `WAPRO_ALLOW_EXTERNAL`, `WAPRO_ALLOW_DANGEROUS` | nie | flagi trybu pełnego, włącza je tylko wartość `1` |

### 5. Rejestracja w kliencie MCP

Serwer uruchamia plik `<folder_aplikacji>\mcp-wapromag.exe` (bez `node`).

**Claude Code** (tryb tylko do odczytu):

```
claude mcp add wapro-mag -s user \
  -e WAPRO_SQL_SERVER=<host> -e WAPRO_SQL_DATABASE=<baza> \
  -e WAPRO_SQL_RO_USER=mcp_wapro_ro -e WAPRO_SQL_RO_PASSWORD=<hasło> \
  -e WAPRO_LICENSE_KEY=<klucz> \
  -- <folder_aplikacji>\mcp-wapromag.exe
```

**Claude Desktop**: w `claude_desktop_config.json` (Ustawienia → Developer → Edit Config):

```json
{
  "mcpServers": {
    "wapro-mag": {
      "command": "<folder_aplikacji>\mcp-wapromag.exe",
      "args": [],
      "env": {
        "WAPRO_SQL_SERVER": "<host>",
        "WAPRO_SQL_DATABASE": "<baza>",
        "WAPRO_SQL_RO_USER": "mcp_wapro_ro",
        "WAPRO_SQL_RO_PASSWORD": "<hasło>",
        "WAPRO_LICENSE_KEY": "<klucz>"
      }
    }
  }
}
```

**Inny klient MCP**: uruchom polecenie serwera (jak wyżej) z transportem stdio i zmiennymi z kroku 4.

Hasło trafia do konfiguracji klienta (u Claude Code `~/.claude.json`). Chroń ten plik jak hasło.

### 6. Sprawdzenie

- Claude Code: nowa sesja, `/mcp` pokazuje `wapro-mag` z 4 narzędziami (tryb pełny: 6).
- Ręcznie, w folderze aplikacji, z ustawionymi zmiennymi środowiskowymi: uruchom `mcp-wapromag.exe`. Na stderr powinno się pojawić `catalog: <liczba> endpoints`, potem serwer czeka na klienta (Ctrl+C kończy).
- Problemy:
  - `Brak klucza licencji` / `Nieprawidłowy klucz licencji` / `Licencja wygasła` / `w tej bazie WAPRO nie ma firmy z tym NIP`: krok 0;
  - `Brak katalogu`: wróć do kroku 3;
  - `Login failed`: sprawdź login, hasło i tryb mieszany SQL Server;
  - `Missing required env vars`: brakuje zmiennej z kroku 4.

## Użycie

Piszesz do asystenta zwykłym językiem, a on sam dobiera narzędzia. Przykłady w trybie tylko do odczytu:

- „Pokaż magazyny firmy SolidnaFirma.” → `wapro_sql` na `Api_V_Warehouse`.
- „Ile faktur sprzedaży wystawiliśmy w październiku i na jaką kwotę brutto?” → `wapro_describe_endpoint` (pole `views`), potem `wapro_sql`.
- „Którzy kontrahenci mają przeterminowane należności powyżej 1000 zł?”
- „Jakie pola przyjmuje endpoint wystawienia faktury i co oznacza błąd INNER_ERROR?” → `wapro_find_endpoints`, `wapro_describe_endpoint`.

Słowniki ID (typy dokumentów, formy płatności, stawki VAT) są w widokach `Api_V_DocumentTypeInvoice`, `Api_V_DocumentTypeWarehouse`, `Api_V_PaymentForm` i `Api_V_TaxRate`. Serwer podpowiada je asystentowi w instrukcjach.

W trybie pełnym (baza testowa) zapis przebiega tak:
1. `wapro_activity`: kolejność endpointów danej czynności.
2. `wapro_describe_endpoint`: pola i ostrzeżenia.
3. `wapro_write` z `dryRun: true`.
4. `wapro_write` bez `dryRun`, czyli commit. Wymaga `WAPRO_ALLOW_WRITE=1`.

Przykład: „Wystaw fakturę dla kontrahenta X na 1 szt. towaru Y w cenie 500 zł netto i ją zatwierdź”. Po commicie 8 kluczowych endpointów (`…Accept`, inwentaryzacja) jest weryfikowanych odczytem. `verified: false` oznacza, że WAPRO zgłosiło sukces, ale dokument się nie zmienił.

## Tryb pełny (tylko baza testowa)

1. Odtwórz kopię bazy produkcyjnej (np. backup `COPY_ONLY`) pod nazwą zawierającą `TEST`, np. `WAPRO_MCP_TEST`.
2. Uruchom `setup-logins.sql` dla tej kopii, a potem login zapisu:
   ```
   sqlcmd -S localhost -E -b -i scripts\setup-write-login.sql -v DB=WAPRO_MCP_TEST RW_PASS=<silne_hasło>
   ```
   - Skrypt przerywa, gdy nazwa bazy nie zawiera `TEST`.
   - Login `mcp_wapro` dostaje `db_datareader`, `db_datawriter`, `EXECUTE` i `VIEW DEFINITION`.
   - Celowo nie dostaje `db_owner`: bazy WAPRO mają `TRUSTWORTHY ON` i właściciela `sa`, więc `db_owner` pozwala przejąć sysadmina całego serwera.
   - Na koniec login jest **wyłączony**. Włączaj go tylko na czas pracy: `sqlcmd -S localhost -E -Q "ALTER LOGIN [mcp_wapro] ENABLE"`, a potem `… DISABLE`. Działa na tym samym serwerze co produkcja.
3. Zarejestruj drugi serwer (np. `wapro-mag-test`) ze zmiennymi `WAPRO_SQL_USER`, `WAPRO_SQL_PASSWORD`, `WAPRO_USER_ID` i `WAPRO_COMPANY_ID`. Do zapisu dodaj `WAPRO_ALLOW_WRITE=1`.
4. `WAPRO_USER_ID` to użytkownik API WAPRO (WAPRO Anywhere), czyli wiersz w `WAPROERP_MANDANT_USER`. Znajdziesz go zapytaniem `SELECT ID_UZYTKOWNIKA FROM WAPROERP_MANDANT_USER`.

Nigdy nie dodawaj loginu zapisu do bazy produkcyjnej. Bez niego produkcja może działać tylko w trybie tylko do odczytu.

## Bezpieczeństwo

- **Tryb tylko do odczytu**: w konfiguracji nie ma loginu zapisu, a w bazie produkcyjnej nie powinien istnieć. Żadna procedura WAPRO nie jest uruchamiana. Ograniczenia `wapro_sql` opisuje sekcja [Narzędzia](#narzędzia).
- **Zapis** (tryb pełny): bez `WAPRO_ALLOW_WRITE=1` commit jest odrzucany („writes disabled”), a `dryRun: true` działa. Commit następuje tylko wtedy, gdy:
  - nie ma błędów;
  - `errorData` jest puste;
  - własna transakcja serwera jest nienaruszona (`trancount` 1, ten sam identyfikator transakcji).
- **Efekty zewnętrzne**: 58 endpointów woła HTTP lub e-mail (SMS, płatności online, kurierzy, KSeF, AI). ROLLBACK ich nie cofa.
  - Klasa `realne` (np. `PackageSend`, `BlmtDocumentSend`, odczyt `PackageReceiveReportGet`): zakaz dry-run i `wapro_read`. Commit wymaga `WAPRO_ALLOW_WRITE=1` i `WAPRO_ALLOW_EXTERNAL=1`.
  - Klasa `warunkowe` (SMS, płatności online): dry-run tylko przy nieaktywnych strażnikach, liczonych przy starcie osobno dla każdej firmy.
  - Klasa `lookup` (NBP, VIES): jak zwykły zapis.
- **Klasy `broken` i `dangerous`**:
  - `broken` to wady producenta; serwer odmawia wywołania.
  - `dangerous`: commit wymaga `WAPRO_ALLOW_DANGEROUS=1`. Dotyczy DDL, usuwania firm i magazynów, `UPDATE` bez `WHERE` oraz zapisu kodu SQL wykonywanego później (atrybuty Genius, `ReportAddOrModify`, `KsefFieldAddOrModify`).
  - Endpointy z `requireFields` wymagają pól, bez których procedura robi coś innego, niż sugeruje nazwa.
- **Walidacja wejścia**: WAPRO skleja część pól do dynamicznego SQL, więc obowiązuje ona w każdym trybie, na każdym poziomie `data` i `filter`:
  - pola `OrderBy`, `SortColumn`, `ColumnName`, `MenuName`, `TableName`, `FilterColumn`, `AtribiuteTable`, `TableKey`, `ValueColumn`: tylko identyfikatory; `Order`: tylko `ASC` lub `DESC`;
  - pola z kodem SQL (`JoinCondition`, `SqlFilter`, `SqlCode`, `SqlFunction`) mogą być niepuste tylko z `WAPRO_ALLOW_DANGEROUS=1`;
  - `Gdpr_AnonymizeData.EntityType`: tylko `Kontrahent`, `Kontakt`, `Pracownik`, `Osoba_powiazana`;
  - `AdditionalTableValueAddOrModify.Item`: klucze tylko identyfikatory, teksty bez `'` i `&`, tylko znaki strony kodowej 1250.
- **Maskowanie**: w wynikach wartości pod kluczami `password`, `haslo`, `token`, `secret`, `apikey` itp. są zastępowane `"***"`.

## Telemetria WAPRO (tryb pełny)

`AP_ZapiszDoLogu` dla użytkownika z `UZYTKOWNIK.TELEMETRIA_ZGODA=1` czyta `sys.dm_exec_connections`. Bez uprawnienia serwerowego każde `…Accept`, także w dry-run, kończy się `INNER_ERROR`, a zapis jest cofany.

Na bazie testowej ustaw `TELEMETRIA_ZGODA=0` dla użytkownika API. Skrypty celowo nie nadają `VIEW SERVER PERFORMANCE STATE`, bo to uprawnienie pokazuje treść zapytań innych sesji.

## Dry-run bywa niepełny

Pokrycie zmierzone na bazie testowej WAPRO Mag 9.20:
- 1404 przebiegi: 801 ok, 560 `wapro_error` (oczekiwana walidacja pustego wejścia), 16 `sql_error`, 27 pominiętych, 0 awarii;
- liczności tabel bez zmian.

W 382 endpointach procedura WAPRO przy błędzie cofa transakcję serwera, więc instrukcje wykonane **po** tym mogą się zapisać mimo dry-run. Wszystkie przypadki z pokrycia są na ścieżkach błędu. Serwer:
- dodaje wtedy ostrzeżenie („cofnęła transakcję serwera MCP”);
- takiej transakcji nie zatwierdza;
- cofa transakcję otwartą przez procedurę po jej ROLLBACK.

Dry-run to podgląd, nie gwarancja. Dlatego tryb pełny jest tylko dla bazy testowej, a produkcja działa w trybie tylko do odczytu.

## Opisy czynności (opcjonalnie)

`wapro_activity` i opisy w `wapro_find_endpoints` korzystają z plików, których paczka nie zawiera:
- `docs/czynnosci/<obszar>.md`: sekcje `## <czynność>` z nazwami endpointów w backtickach, w kolejności wywołań;
- `docs/czynnosci/opisy/<obszar>.json`: `{ "<op>": "<jedno zdanie, co robi endpoint>" }`.

Bez nich serwer działa, tylko `wapro_activity` nic nie zwraca.

## Licencja

Oprogramowanie płatne, wszelkie prawa zastrzeżone (plik [LICENSE](LICENSE)). Korzystanie wymaga zakupionego klucza licencji: 2 900 zł netto jednorazowo (licencja wieczysta na NIP firmy, 12 miesięcy aktualizacji), przedłużenie aktualizacji 580 zł netto rocznie. Z kluczem wolno uruchamiać aplikację bez zmian, w wersjach wydanych w okresie aktualizacji. Kod źródłowy nie jest udostępniany. Bez pisemnej zgody właściciela nie wolno modyfikować kodu, obchodzić sprawdzania licencji, udostępniać klucza, kopiować kodu do innych projektów ani go rozpowszechniać.
