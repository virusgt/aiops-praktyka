# Lab05 — dziwne momenty w Kantynie (namespace `demo`)

Czas: Europe/Warsaw (CEST, UTC+2).

```text
13.10.2026 13:01–14:40 | kolejka workera do ~11 020 widocznych wiadomości (norma 0–29), processing_delay_ms=2000 od 13:01:17 do 13:46:02 | max(worker_queue_visible_messages{namespace="demo"})
13.10.2026 11:32–12:15 | p95 POST /api/orders ~2,42 s (norma ~40 ms), payments latency_ms=1200 + jitter 600 od 11:30 do 12:11 | histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="demo", job="kantyna-orders-api", method="POST", route="/api/orders"}[5m])))
13.10.2026 10:32–11:05 | błędy HTTP 5xx orders-api do ~7 % (norma 0 %), ~406 odpowiedzi 5xx, 423 wpisy ERROR w Loki | 100 * sum(rate(http_requests_total{namespace="demo", job="kantyna-orders-api", status=~"5.."}[5m])) / sum(rate(http_requests_total{namespace="demo", job="kantyna-orders-api"}[5m]))
```

## 1. Nagromadzenie wiadomości w kolejce workera — 13.10.2026, 13:01–14:40

Status: zweryfikowane ręcznie.

### Fakty potwierdzone metryką

- Kolejka zaczęła rosnąć wkrótce po 13:01. Najwyższa wartość to ok. **11 020** widocznych wiadomości.
- Normalnie kolejka ma **0–29** wiadomości: 0–1 poza szczytem, do ~29 w szczycie obiadowym.
- Około **14:40** kolejka wróciła do zera.
- Metryka `worker_queue_visible_messages` (gauge) to **przybliżona** liczba widocznych wiadomości w kolejce SQS (opis metryki: „Approximate number…”). Worker odczytuje ją z atrybutu SQS `ApproximateNumberOfMessages` (`app/worker/app/main.py`), który z definicji jest przybliżony, a Prometheus ją próbkuje, więc maksimum to wartość orientacyjna. Nie obejmuje wiadomości, które worker właśnie przetwarza, ani wiadomości w DLQ.

PromQL:

```promql
max(worker_queue_visible_messages{namespace="demo"})
```

### Potwierdzenie w logach (Loki)

- 13:01:17: worker loguje `flags updated` z `processing_delay_ms=2000`.
- 13:46:02: worker loguje `flags updated` z `processing_delay_ms=0`.
- Logi pokrywają się w czasie z metryką: kolejka rośnie od włączenia opóźnienia i spada do zera po jego wyłączeniu, bo worker nadrabia zaległości.

LogQL:

```logql
{namespace="demo", app="worker"} |= "flags updated"
```

### Hipoteza przyczyny (niepotwierdzona)

Kolejka urosła prawdopodobnie przez wstrzyknięte flagą opóźnienie 2 s na każdą wiadomość. Przy jednym podzie workera taki worker nie nadąża za napływem zamówień. Logi i metryka zgadzają się w czasie, ale związek przyczynowy to nadal hipoteza. Nie wiemy też, kto zmienił flagę.

## 2. Wzrost p95 czasu odpowiedzi POST /api/orders (A2) — 13.10.2026, ok. 11:32–12:15

Status: zweryfikowane ręcznie.

### Fakty potwierdzone metryką

- p95 czasu odpowiedzi POST /api/orders wzrosło z typowych ok. **40 ms** do ok. **2,42 s**, czyli mniej więcej 60 razy.
- Prometheus pokazuje gwałtowny skok na początku okna i powrót do normy (~40 ms) na jego końcu.
- p95 to **estymacja z histogramu** (`histogram_quantile` interpoluje liniowo w obrębie kubełka), a nie pomiar konkretnego żądania. Wartość 2,42 s leży w kubełku 1,0–2,5 s, więc pewne jest tylko, że p95 mieści się w tym przedziale. Na początku i końcu okna wynik dodatkowo rozmywa okno `rate(...[5m])`.

PromQL:

```promql
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="demo", job="kantyna-orders-api", method="POST", route="/api/orders"}[5m])))
```

### Potwierdzenie w logach (Loki)

- 11:30:24 i 11:30:54: oba pody payments logują `flags updated` z `latency_ms=1200` i `latency_jitter_ms=600`.
- 12:10:39 i 12:11:09: oba pody payments logują `flags updated` z `latency_ms=0` i `latency_jitter_ms=0`.
- Zmiany flag poprzedzają o kilka minut początek i koniec wzrostu p95, co pasuje do 5-minutowego okna `rate`.

LogQL:

```logql
{namespace="demo", app="payments"} |= "flags updated"
```

### Hipoteza przyczyny (niepotwierdzona)

Prawdopodobnie wzrost p95 w orders-api to skutek opóźnienia wstrzykniętego flagą w payments. orders-api przy każdym POST /api/orders wywołuje payments (`POST /pay`) **synchronicznie** i czeka na odpowiedź, zanim zapisze zamówienie i odpowie klientowi. Czas odpowiedzi orders-api zawiera więc cały czas obsługi płatności. Przy `latency_ms=1200` i `latency_jitter_ms=600` płatność trwa ok. 1,2–1,8 s, a z czasem pracy samego orders-api (baza, kolejka) daje to p95 w kubełku 1–2,5 s.

Hipotezę wspiera zgodność w czasie i rząd wielkości opóźnienia. Nie potwierdzono jednak bezpośrednio, np. trace'em w Tempo, że to spany wywołania `/pay` wydłużały żądania. Nie wiemy też, kto zmienił flagę.

## 3. Skok błędów HTTP 5xx w orders-api (A1) — 13.10.2026, ok. 10:32–11:05

Status: zweryfikowane ręcznie.

### Fakty potwierdzone w Prometheusie

- Odsetek odpowiedzi HTTP 5xx w orders-api wzrósł do ok. **7 %**. To odczyt orientacyjny z wykresu; najwyższy punkt w kroku 5 min to ok. 6,9 %. Poza tym oknem odsetek wynosi **0 %**.
- Po ok. 11:05 odsetek wraca do 0 %.
- W oknie przybyło ok. **406** odpowiedzi 5xx, z czego ok. 316 to GET i ok. 91 to POST. Wszystkie mają etykiety `route="unmatched", status="500"`.

PromQL:

```promql
100 * sum(rate(http_requests_total{namespace="demo", job="kantyna-orders-api", status=~"5.."}[5m])) / sum(rate(http_requests_total{namespace="demo", job="kantyna-orders-api"}[5m]))
```

Liczba odpowiedzi 5xx (pomocniczo):

```promql
sum by (method, route, status) (increase(http_requests_total{namespace="demo", job="kantyna-orders-api", status=~"5.."}[3h]))
```

### Fakty potwierdzone w Loki

- W badanym zakresie Loki zawiera **423** wpisy na poziomie ERROR z orders-api.
- Wszystkie mają komunikat `Unhandled error while processing request`.
- Traceback zawiera `runtime.raise_synthetic_error` (wywoływane z `observability_middleware`) oraz wyjątek `UpstreamUnavailableError` („order pipeline dependency returned an unexpected response”).
- W logach widać prawdziwe trasy żądań (`/api/menu`, `/api/menu/search`, `/api/orders`), a w metryce wszystkie błędy mają `route="unmatched"`. Wynika to prawdopodobnie z tego, że wyjątek pada w middleware jeszcze przed dopasowaniem trasy, więc w metryce trasa nie jest znana.

LogQL:

```logql
{namespace="demo", app="orders-api"} | json | level="ERROR"
```

### Hipoteza przyczyny (niepotwierdzona)

Błąd był **sztucznie wstrzyknięty**, prawdopodobnie flagą `error_rate` w orders-api (mechanizm awarii ćwiczeniowych Kantyny). Przemawia za tym nazwa funkcji `raise_synthetic_error` w tracebacku i to, że błędy dotyczą wszystkich tras naraz, a nie jednej zależności. Komunikat „upstream unavailable” to treść syntetycznego wyjątku, a nie dowód awarii prawdziwej zależności. W logach orders-api z 4 dni nie znaleziono jednak wpisów o zmianie flag (inaczej niż w payments i workerze), więc nie widać, kiedy flagę włączono i wyłączono, ani kto to zrobił.

### Ograniczenia pomiaru

- **423 wpisy ERROR to nie to samo co 423 żądania 5xx.** Jedno żądanie może wygenerować więcej niż jeden wpis ERROR. Część wpisów może dotyczyć błędów, które nie skończyły się odpowiedzią 5xx. Odwrotnie, odpowiedź 5xx nie musi mieć wpisu ERROR. Liczby 406 i 423 są zbliżone, co wspiera związek, ale nie przekładają się 1:1.
- **Zakresy obu liczb się różnią.** Wynik 406 pochodzi z `increase(...)` w oknie kończącym się ok. 11:00, a 423 z zapytania Loki w zakresie ok. 10:30–11:10. `increase()` dodatkowo ekstrapoluje i zwraca wartości ułamkowe.
- **Odsetek 5xx w mianowniku zawiera ruch techniczny** (`/healthz`, `/readyz`, `/metrics`), więc zaniża udział błędów w ruchu użytkowników.
- **Granice okna są przybliżone** z dokładnością do kroku 5 min i okna `rate(...[5m])`. Przed ok. 10:32 szeregów 5xx w ogóle nie było (brak szeregu, a nie zmierzone zero).
