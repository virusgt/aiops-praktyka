# Przegląd PR #1

- **PR:** #1 — web: nowa stopka
- **Werdykt:** POPRAW

## 3 najważniejsze powody
1. Opis PR-a nie ma sekcji „Jak przetestowałem”. Testy są opisane w sekcji `## Testy`, a to nie spełnia nowej reguły z REVIEW.md.
2. Diff nie łamie żadnej reguły blokującej: zmienia jedną linię tekstu w HTML (`app/web/html/index.html:64`) i nie zawiera sekretów, obrazów, `permissions` ani akcji GitHub.
3. Opis mówi, co i dlaczego się zmienia, a checki CI przechodzą (testy i helm lint: pass, build i deploy: skipping).

## Blokery
brak

## Uwagi do poprawy
- Opis PR-a — reguła „opis PR-a nie ma sekcji "Jak przetestowałem"”: wystarczy zmienić nagłówek `## Testy` na `## Jak przetestowałem`. Treść sekcji już opisuje wykonane i planowane testy.
- Drobna uwaga poza regułami: test ręczny w przeglądarce (`docker compose up`) jest w opisie oznaczony jako planowany, ale niewykonany. Warto go zrobić przed merge'em.

## GitHub Actions
| Check | Stan |
|---|---|
| helm lint | pass |
| test orders-api (Python 3.12, uv) | pass |
| test payments (Go 1.23) | pass |
| build ${{ matrix.service }} | skipping |
| deploy → ${{ vars.K8S_NAMESPACE }} | skipping |

## Uzasadnienie
Sama zmiana jest bezpieczna i kosmetyczna, a CI przechodzi, ale opis nie spełnia reguły o sekcji „Jak przetestowałem”. Po zmianie nagłówka sekcji PR kwalifikuje się do MERGE.
