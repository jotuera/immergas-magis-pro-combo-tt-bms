# Immergas Magis Pro / Combo (Audax) — Modbus BMS przez ESPHome (M5 Atom)

🇬🇧 [English](README.md) | 🇵🇱 Polski

<p align="center">
  <img src="https://img.shields.io/github/v/tag/jotuera/immergas-magis-pro-combo-tt-bms?label=version&color=blue" alt="Wersja">
  <img src="https://img.shields.io/badge/ESPHome-2026.9%2B-41BDF5.svg" alt="ESPHome 2026.9+">
  <img src="https://img.shields.io/badge/platform-ESP32-41BDF5.svg" alt="ESP32">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-green.svg" alt="Licencja: MIT"></a>
</p>

Odczyt i sterowanie pompą ciepła **Immergas Magis Pro / Combo** (Audax) po magistrali **T-/T+ (BMS) Modbus RS485** za pomocą **M5Stack Atom + ESPHome**, zintegrowane z Home Assistantem.

To projekt społecznościowego reverse-engineeringu. Immergas nie publikuje mapy rejestrów Modbus dla tej magistrali, więc poniższa mapa powstała przez skanowanie, porównanie z instrukcją instalatora Magis Combo, logowanie na żywo cykli pracy i zestawienie z magistralą Samsung NASA (F1/F2).

Projekty siostrzane (ten sam autor, docelowo mają działać razem na jednym urządzeniu):
- [immergas-magis-pro-combo-dd-modbus](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus) — emulator Dominusa / panelu strefy na magistrali D+/D-
- [immergas-magis-pro-combo-f1f2-nasa](https://github.com/jotuera/immergas-magis-pro-combo-f1f2-nasa) — odczyt magistrali Samsung NASA F1/F2

## Sprzęt

- **M5Stack Atom Lite** (ESP32) + transceiver **RS485** (np. MAX3485 / moduł RS485 M5)
- Podłączenie: Immergas **T+ / T-** → RS485 **A / B** (zamień A/B, jeśli dostajesz błędy CRC)
- Magistrala: **Modbus RTU, slave 11, 9600 8N2**, 2-przewodowa (instrukcja Immergas podaje 8N2; odbiorniki 8N1 też działają)
- Testowane na: **Magis Combo 9 Plus V2** (Audax 9, 9 kW, firmware D91 = 6.0) i **Magis Pro V2** (potwierdzone przez społeczność)
- Atom jest Modbusowym **masterem**; pompa ciepła jest slave'em 11.

## Co nowego w v2.0.0

- **Konfiguracja modularna.** Pojedynczy 1300-liniowy YAML podzielony jest na pakiety (`packages/tt_*.yaml`), które ESPHome pobiera z GitHuba. Kopiujesz tylko krótki plik główny i wybierasz, które funkcje chcesz. Osobny plik `-beta.yaml` zniknął; skaner PDU jest teraz opcjonalnym pakietem (`tt_scanner.yaml`).
- **Języki:** nazwy encji i teksty pochodzą z jednego pliku językowego: `packages/lang/tt_en.yaml` (angielski) lub `packages/lang/tt_pl.yaml` (polski). Każdą pojedynczą nazwę można nadpisać w `substitutions:`.
- **Szacowane zużycie gazu** (`tt_gas.yaml`): czas pracy kotła × skalibrowane m³/h dla CO i CWU, jako licznik `total_increasing` m³ do dashboardu Energy w Home Assistant.
- **Nowo rozszyfrowane:** 4554 = **D73 temperatura tłoczenia sprężarki**, 4585 = **D72 temperatura sprężarki** (obie zgodne z wartościami Samsung NASA co do 0,1°C), 4586 = **stopień mocy sprężarki 0-6** (nie flaga wł./wył.).
- **Nowe kody stanów:** D99 7 / D97 336 / D98 236 (grzanie tylko kotłem, pompa ciepła wykluczona przez I10), D99 44 (koniec CWU na pompie ciepła), D99 61 = **CWU na pompie ciepła** (nie "wspomaganie solarne"), D98 215 = grzanie kotłem (cykl).
- "Heating active" / "Cooling active" nie używają już device_class `heat` / `cold`, więc pokazują on/off tak jak "DHW active".
- Nazwy plików i identyfikatory zaczynają się od `tt_`, więc to repo nigdy nie koliduje z projektami D+/D- i F1/F2.

⚠️ **Aktualizacja z 1.x:**
- **Przemianowanie:** trzy odczyty zostały przemianowane i pojawią się jako **nowe encje**: `Refrigerant hot-side temperature (cand.)` → `D73 Compressor discharge temperature`, `Refrigerant circuit 4585 (cand. temp D72-74)` → `D72 Compressor temperature`, `Compressor running` → `Compressor power step`. Usuń stare w HA.
- **Bez zmian:** wszystkie pozostałe nazwy encji są takie same, więc ich historia jest kontynuowana.
- **Przełączenie:** zastąp swój YAML nowym plikiem głównym. Przepisz do niego swoje ustawienia Wi-Fi / stałego IP i zachowaj tę samą `device_name`.

## Struktura projektu

```
immergas-magis-pro-combo-tt-bms.yaml   plik główny: urządzenie, Wi-Fi, API + lista pakietów (jedyny plik, który kopiujesz)
packages/
  tt_core.yaml          UART, Modbus, dwa kontrolery odpytywania, przełącznik "BMS connection"     (wymagany)
  tt_status.yaml        tryb pracy, D97/D98/D99, kod i tekst błędu kotła, A11, D61, D91,            (wymagany)
                        zegar wewnętrzny, "Heating/Cooling/DHW active"
  tt_circuit.yaml       zasilanie/powrót D20/D08, zasilanie kotła (3029), D14, D06, D04,             (wymagany)
                        zawór 3-drogowy D22
  tt_heat_pump.yaml     D24, częstotliwość D71, D72/D73/D74, wentylator D76, EEV D77, stopień mocy sprężarki
  tt_dhw.yaml           D03 temperatura zasobnika, water_heater CWU (2095), T02
  tt_heating.yaml       sonda zewnętrzna R01, limity strefy 1 R04/R05/R12/R13, włączenie termostatu strefy 1 (2010)
  tt_integration.yaml   menu I: I01, I02, I03, I04, I05, I07, I09
  tt_installer.yaml     P15, A12, prędkość pompy A03/A04, T04, T05, T07
  tt_computed.yaml      Delta T, moc cieplna, średnia temperatura obiegu, thermal lift, powrót PC - zasilanie kotła
  tt_gas.yaml           szacowane zużycie gazu (wymaga tt_status + tt_circuit)
  tt_diagnostics.yaml   nierozszyfrowane rejestry + nieudokumentowane zapisywalne rejestry 0/1
  tt_scanner.yaml       opcjonalny aktywny skaner PDU
  lang/tt_en.yaml       nazwy/teksty angielskie
  lang/tt_pl.yaml       nazwy/teksty polskie
```

Żeby usunąć funkcję, skasuj jej linię z `packages:` w pliku głównym. `tt_core`, `tt_status` i `tt_circuit` są zawsze potrzebne.

## Instalacja

1. Skopiuj `immergas-magis-pro-combo-tt-bms.yaml` do ESPHome (dashboard → nowe urządzenie → wklej) oraz `secrets.yaml.example` → `secrets.yaml` (Wi-Fi, hasło fallback AP).
2. Wybierz linię języka (`packages/lang/tt_en.yaml` lub `tt_pl.yaml`). Jeśli Twoje piny RS485 są inne, ustaw `tt_uart_tx_pin` / `tt_uart_rx_pin` / `tt_uart_de_pin` w `substitutions:`.
3. Wgraj przez **ESPHome 2026.9 lub nowszy**, potem dodaj urządzenie w Home Assistant.
4. Żeby zaktualizować później, zmień `ref: v2.0.0` na nowszy tag (albo `main` dla wersji rozwojowej).

## Co robi

- **Odczytuje:** temperatury zasilania/powrotu/CWU/zewnętrzną, **wyliczoną nastawę zasilania (D04)**, częstotliwość i stopień mocy sprężarki, temperatury sprężarki/tłoczenia/parownika, pozycję EEV, obroty wentylatora zewnętrznego, pozycję zaworu 3-drogowego, status pompy ciepła/kotła/systemu, kod błędu + opis, model systemu (D61), wersję firmware (D91).
- **Steruje:** trybem pracy (2000), nastawą CWU (2095), sondą zewnętrzną krzywej grzewczej (R01), limitami grzania/chłodzenia strefy 1 (R04/R05/R12/R13), minimalną/maksymalną prędkością pompy obiegowej (A03/A04), termostatem strefy 1, **integracją gaz/pompa ciepła (I01–I09)**, T02, T04, T05, T07, anty-legionellą P15, odpowietrzaniem systemu A12.
- **Wylicza:** Delta T, średnią temperaturę obiegu, thermal lift, różnicę powrót pompy ciepła − zasilanie kotła, **szacowane zużycie gazu**.
- Przełącznik połączenia BMS (start/stop odpytywania).

## Szacowane zużycie gazu

Urządzenie nie raportuje zużycia gazu, więc `tt_gas.yaml` liczy czas pracy kotła gazowego i mnoży go przez średnie m³/h, z osobnymi stawkami dla CO i CWU.
- **Kiedy liczy:** kocioł pracuje = D99 62 (CWU na gazie), D98 215/236 albo D99 7 (CO przez kocioł). Przy grzaniu (D99 6/8) liczy też, gdy zasilanie kotła jest co najmniej 8 K wyższe od powrotu pompy ciepła. Normalne grzanie pompą ciepła (D98 252) nie jest liczone.
- **Kalibracja:** odczytaj swój licznik gazu na początku i końcu tygodnia. Podziel realne m³ przez przyrost `Gas estimated consumption`, potem pomnóż obie stawki (`Gas consumption CH/DHW (calibration)`) przez ten współczynnik.
- **Dashboard Energy:** dodaj `Gas estimated consumption` w Ustawienia → Dashboardy → Energia → Zużycie gazu. Jeśli licznik kiedykolwiek skoczy albo zresetuje się, sprawdź statystykę kosztu gazu (Narzędzia deweloperskie → Statystyki) pod kątem skoku.

## ⚠️ Ważne zastrzeżenia

- **Tryb pracy: wygrywa ten, kto go ciągle wymusza.** *Emulowany* panel strefy na D+/D- stale wymusza swój tryb i cofa każdy tryb zapisany przez T-/T+ (rejestr 2000). Z *prawdziwym* panelem Dominus (firmware 6.0), jednostka trzyma tryb zapisany przez T-/T+ jako polecenie BMS. Wtedy to ona cofa zmiany trybu zrobione z klawiatury albo z Dominusa. Z działającym masterem T-/T+, zmieniaj tryb z Home Assistanta.
- **Wejście termostatu 40-1/41 wymaga panelu strefy 1 na D+/D-.** Na firmware 6.0, przy działającym masterze BMS, styk termostatu na 40-1/41 uruchamia grzanie **tylko** gdy na D+/D- jest obecny panel strefy 1: prawdziwy panel Dominus (A31 = RPT) albo [emulator dd-modbus](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus). Bez niego styk jest ignorowany, a rejestr 2010 nie pomaga (dwóch użytkowników na fw 4.3 i 6.0). Zgłoszenia że 2010 działa pochodzą z firmware 8.3+. Zob. [issue #1](https://github.com/jotuera/immergas-magis-pro-combo-tt-bms/issues/1).
- **Niezamontowane sondy czytają `-32768` (0x8000),** np. rejestr 3029 (zasilanie kotła) na Magis Pro bez kotła gazowego. Wszystkie sensory temperatury mapują to na *unknown*.

## Fakty o magistrali

- **Kody funkcji:** jednostka odpowiada **wyłącznie na Modbus function 0x03**. Funkcje 0x01, 0x02 i 0x04 są odrzucane (illegal function). Zapisy używają 0x06.
- **Banki rejestrów:** **2xxx** (konfiguracja/sterowanie), **3xxx** (sensory), **4xxx** (pompa ciepła/czynnik), **6xxx** (parametry/statusy). Cały zakres **0–19999** został przeskanowany; nic nie odpowiada powyżej 6510.
- **Brak interfejsu Victrix / Magis M TOP:** niskie rejestry BMS z instrukcji Victrix i Magis M TOP nie są dostępne na Magis V2 (fw 6.0). 512 jest **tylko do odczytu** (zapis zwraca wyjątek 2), 256 i rejestry wejściowe 0x04 nie istnieją, a 61440–61442 (konfiguracja Modbus) zwracają wyjątek 2. Na tej magistrali nie ma żądania ciepła BMS ani "trybu cichego" BMS.
- **Stała mapa:** wpisanie PIN-u instalatora **nie** odsłania więcej rejestrów. Punkty krzywej grzewczej (R02/R03) i większość parametrów menu nie są na tej magistrali.
- **Banki menu:** bank **44xx** zawiera ustawienia integracji/instalatora (menu I, T, P). Bank **65xx** zawiera odczyty informacyjne (menu D: D97–D99, D61, D91, zegar wewnętrzny).
- **Ustawienia niewidoczne na T-/T+** (zmienione na jednostce, żaden rejestr się nie rusza): I08, I10, P07 (ale D06 czyta *po* korekcie P07), P11, P13, P21, P23, P24, T09, T23, **funkcja nocna U11** (idzie do jednostki zewnętrznej jako limit częstotliwości 30% na D+/D-, nie tutaj), A05, A18, A31, A41, menu serwisowe M (M13, M15–M17). Ustawienia zdalnego panelu strefy są na magistrali D+/D-.
- **Menu serwisowe:** wejście w menu **M** ustawia kod błędu **193** (E193 "tryb testowy") w rejestrze 2100, dopóki go nie opuścisz.
- **Nazewnictwo:** "PDU N" u Immergasa = adres rejestru trzymającego N.

## Mapa rejestrów (znane)

| Rej | Kod | Opis | Skala |
|---|---|---|---|
| 512 | — | Tylko do odczytu na Magis V2 (zawsze 0 od października 2026; raz widziano 280/2000 w czerwcu) | — |
| 2000 | — | Tryb pracy (0=standby, 1=lato/CWU, 2=chłodzenie, 3=zima) | RW |
| 2010 | — | Termostat/żądanie grzania Strefy 1 (bit0) — zob. zastrzeżenie wyżej | RW |
| 2095 | D05 | Nastawa CWU | ×0,1°C, RW |
| 2100 | — | Kod błędu (0 = brak, inaczej kod E) | R |
| 3000 | D20 | Temperatura zasilania systemu | ×0,1°C |
| 3001 | D08 | Temperatura powrotu wody z pompy ciepła | ×0,1°C |
| 3002 | D06 | Temperatura zewnętrzna | ×0,1°C |
| 3003 | D04 | **Wyliczona nastawa zasilania** z krzywej grzewczej; 0 = brak żądania ciepła | ×0,1°C |
| 3016 | D03 | Temperatura zasobnika CWU | ×0,1°C |
| 3027 | — | Kandydat: 1 = pompa ciepła wykluczona / grzanie tylko kotłem (widziane raz, test I10) | 0/1 |
| 3028 | D22 | Zawór 3-drogowy generatora (1 = CWU, 0 = CO) | 0/1 |
| 3029 | — | **Temperatura zasilania generatora dodatkowego (kocioł gazowy)**; `-32768` na jednostkach bez kotła | ×0,1°C |
| 3054 | D14 | Przepływ pompy obiegowej (0 gdy pompa stoi) | l/h |
| 3056 | D24 | Temperatura czynnika w obiegu chłodniczym | ×0,1°C |
| 4201 | R01 | Sonda temperatury zewnętrznej dla krzywej grzewczej: 0 = OFF (stała zadana zasilania), 1 = OU (sonda jednostki zewnętrznej), 2 = IU (sonda na jednostce wewnętrznej) | enum, RW |
| 4202 | A11 | Model jednostki zewnętrznej | kod (9 = 9 kW) |
| 4350 | R05 | Minimalna temperatura CO Strefy 1 | °C, RW |
| 4351 | R04 | Maksymalna temperatura CO Strefy 1 | °C, RW |
| 4356 | R13 | Maksymalna temperatura chłodzenia Strefy 1 | °C, RW |
| 4357 | R12 | Minimalna temperatura chłodzenia Strefy 1 | °C, RW |
| 4402 | T02 | Termostat CWU (różnica włączenia CWU) | °C, RW |
| 4403 | P15 | Funkcja anty-legionella | 0/1, RW |
| 4411 | T04 | (nieopisane w żadnej instrukcji Immergas) | RW |
| 4451 | I05 | Tryb aktywacji integracji (0 = AU automatyczny, 1 = MA ręczny) | RW |
| 4453 | I02 | Włączenie integracji systemu (CO) (1 = AL, 0 = OFF) | 0/1, RW |
| 4454 | I01 | Włączenie integracji CWU (1 = AL, 0 = OFF) | 0/1, RW |
| 4455 | I09 | Temperatura aktywacji CWU | °C, ze znakiem, RW |
| 4468 | I04 | Maksymalny czas oczekiwania CO | min, RW |
| 4469 | I03 | Maksymalny czas oczekiwania CWU | min, RW |
| 4479 | I07 | Pasmo aktywacji | °C, RW |
| 4551 | D74 | **Temperatura parownika (wymiennik zewnętrzny)** (= NASA 0x8218) | ×0,1°C, ze znakiem |
| 4554 | D73 | **Temperatura tłoczenia sprężarki** (= NASA 0x820A) | ×0,1°C |
| 4557 | D77 | Pozycja zaworu rozprężnego (EEV) (2000 = w pełni otwarty w spoczynku) | 0–2000 |
| 4558 | D71 | **Częstotliwość sprężarki** | Hz |
| 4585 | D72 | **Temperatura sprężarki** (góra, = NASA 0x8280) | ×0,1°C |
| 4586 | — | **Stopień mocy sprężarki** 0–6 (25 Hz = 1, 30 = 2, 43 = 3, 51 = 4, 58 = 5, 61 = 6) | stopień |
| 4587 | D76 | Obroty wentylatora jednostki zewnętrznej (kandydat) | obr/min |
| 6000 | T05 | Minutnik włączeń CO | RW |
| 6002 | T07 | Opóźnienie żądania z termostatu pokojowego (TA): 0-24 = 0-240 s co 10 s (wartość z panelu) | ×10 s, RW |
| 6010 | A03 | Minimalna prędkość pompy obiegowej | %, RW |
| 6011 | A04 | Maksymalna prędkość pompy obiegowej | %, RW |
| 6013 | A12 | Odpowietrzanie systemu (automatyczne po włączeniu zasilania) | 0/1, RW |
| 6500 | D97 | Status żądania pompy ciepła — zob. niżej | 0–999 |
| 6501 | D98 | Status żądania generatora cieplnego — zob. niżej | 0–999 |
| 6502 | D99 | Stan systemu — zob. niżej | 0–999 |
| 6505 | D91 | Wersja oprogramowania P.C.B. (60 = 6.0, 83 = 8.3) | ×0,1 |
| 6506/6507 | D140/D141 | Godzina/minuta wewnętrznego RTC (tylko do odczytu) | h / min |
| 6510 | D61 | Model systemu: 1 = MP, 2 = MCI, 3 = MCP, 4 = MPH, 5 = MCH, 6 = MCPH | kod |

Wszystkie ustawienia RW w wierszach 44xx/60xx zostały zweryfikowane w obie strony: zmiana na klawiaturze przesuwa rejestr, a zapis z Home Assistanta zmienia wartość na jednostce.

**Żądanie pompy ciepła D97:** 300 standby · 332/390 grzanie · 352 grzanie (żądanie termostatu) · 322 CWU na pompie ciepła (fw 4.3) · 342 żądanie chłodzenia · 336 wykluczona przez próg I10 (grzeje kocioł).
**Żądanie generatora cieplnego D98:** 200 standby · 215 grzanie kotłem (cykl) · 232 grzanie · 252 grzanie (żądanie termostatu, kocioł nie pali) · 236 grzanie tylko kotłem.
**Stan systemu D99:** 0 / 82 standby · 1 żądanie chłodzenia · 6 żądanie grzania · 7 grzanie tylko kotłem (pompa ciepła wykluczona przez I10) · 8 cykl grzania · 41 dobieg pompy po CO (~3 min) · 44 koniec CWU na pompie ciepła (~3 min) · 45 koniec CWU (~1 min) · 61 CWU na pompie ciepła · 62 CWU na kotle gazowym (Combo) · 64 CWU grzałką elektryczną. Kody zaobserwowane na fw 6.0. Na fw 4.3 zwykłe CWU było raportowane jako 61.

## 🔎 Nieznane rejestry — pomoc mile widziana!

Wciąż nierozpoznane:
- **Zapisywalne 0/1:** `2102, 4449, 4600` (wszystkie 0).
- **Ustawienie:** `4452` (251 / 236 / 2 na trzech jednostkach).
- **Zawsze zero:** `512, 3061, 4588, 6014, 6503, 6504` (512, 3061 i 6504 poruszyły się raz w czerwcu, może podczas chłodzenia).

Jeśli potrafisz skojarzyć którykolwiek z nich ze stanem pracy, **załóż issue albo PR**.

## 🧪 Skaner PDU (pakiet opcjonalny)

Odkomentuj `packages/tt_scanner.yaml` w pliku głównym:

1. Włącz **`PDU scanner - scanner mode`** (to wstrzymuje normalne odpytywanie).
2. Ustaw **`PDU scanner - from` / `to`** (0–65535) i naciśnij **`Scan range (from-to)`**, albo użyj gotowego przycisku.
3. Każdy adres jest odczytywany funkcją 0x03 (albo 0x04 z przełącznikiem *Input function*). Każda odpowiedź pojawia się jako `OK PDU <N> = <wartość>` w `PDU scanner - last result` i w logu ESPHome.
4. Wyłącz tryb skanera, żeby wrócić do normalnego odpytywania.

Zrzut z instalacji **wielostrefowej** albo z innym firmware to najbardziej przydatna rzecz, jaką możesz udostępnić.

## Jedno urządzenie na kilka magistral

Pakiety `tt_` mogą działać na jednym ESP32 razem z [dd-modbus](https://github.com/jotuera/immergas-magis-pro-combo-dd-modbus) i [f1f2-nasa](https://github.com/jotuera/immergas-magis-pro-combo-f1f2-nasa): listy pakietów stoją obok siebie pod jednym `packages:`, a sekcja urządzenia (esp32, wifi, api, logger) jest jedna. Każda magistrala ma własny UART i piny (`tt_uart_tx_pin` / `tt_uart_rx_pin` / `tt_uart_de_pin`); ESP32 ma trzy sprzętowe UART-y, więc logger trzeba z nich zdjąć (`logger: baud_rate: 0` albo `hardware_uart: USB_SERIAL_JTAG` na ESP32-S3). Żeby odróżnić magistrale w Home Assistant, ustaw prefiks nazw, np. `tt_name_prefix: "Immergas TT"` → *Immergas TT - D20 Temperatura zasilania instalacji*

## Uwagi

- **Opisy błędów:** 114 opisów błędów kotła to oficjalne teksty Immergas (angielski i polski, z etykiet aplikacji Dominus).
- **Znaczenie rejestrów:** znaczenia rejestrów Modbus to reverse-engineering autora; poprawki mile widziane.
- **Przemianowanie tworzy nowe encje:** zmiana nazwy encji tworzy nową encję w Home Assistant (ID encji pochodzi z nazwy), a historia zostaje przy starej. To samo dzieje się przy zmianie pliku językowego.
- **Tłumaczenia:** zob. [TRANSLATING.md](TRANSLATING.md).

## Zastrzeżenie

Zrewersowane na potrzeby **interoperacyjności / użytku prywatnego**. Niezwiązane z Immergasem ani Samsungiem, ani przez nich nieautoryzowane. Zapisywanie do nieznanych/serwisowych rejestrów może zmienić zachowanie urządzenia — używasz na własną odpowiedzialność.

## Licencja

MIT — zob. [LICENSE](LICENSE).
