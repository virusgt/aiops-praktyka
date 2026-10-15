# Przegląd PR

- **PR:** #1 — web: nowa stopka
- **Werdykt:** MERGE

## 3 najważniejsze powody
1. Diff zmienia tylko jedną linię tekstu stopki w `app/web/html/index.html:64`. Nie ma w nim sekretów, obrazów, workflow ani zmian `resources`.
2. Wszystkie checki CI, które się uruchomiły, przeszły: helm lint, test orders-api, test payments.
3. Opis PR-a mówi, co się zmienia (nowy tekst stopki, godziny bez zmian) i dlaczego (czytelniejszy, bardziej przyjazny komunikat).

## Blokery
Brak.

## Uwagi do poprawy
Brak naruszeń reguł z REVIEW.md.

Informacyjnie: w opisie PR-a ręczny test w przeglądarce (`docker compose`, http://localhost:8088) jest oznaczony jako jeszcze niewykonany. REVIEW.md tego nie wymaga, więc nie wpływa to na werdykt.

## GitHub Actions
| Check | Stan |
|---|---|
| helm lint | pass (6s) |
| test orders-api (Python 3.12, uv) | pass (13s) |
| test payments (Go 1.23) | pass (17s) |
| build ${{ matrix.service }} | skipping |
| deploy → ${{ vars.K8S_NAMESPACE }} | skipping |

Build i deploy są pominięte. Opis PR-a mówi, że te kroki uruchamiają się tylko po pushu do `main`; przegląd nie sprawdzał tego w `ci.yml`.

## Uzasadnienie
To kosmetyczna zmiana jednej linii HTML. Checki testów i lint są zielone, a opis jest kompletny. Żadna reguła z REVIEW.md nie jest naruszona, więc PR można zmergować.
