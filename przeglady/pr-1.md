# Przegląd PR #1

- **PR:** #1 — web: nowa stopka
- **Werdykt:** MERGE

## 3 najważniejsze powody
1. Zmiana ogranicza się do jednej linii tekstu w HTML (`app/web/html/index.html:64`): bez sekretów, obrazów, workflow ani zasobów Kubernetes.
2. Wszystkie wymagane checki CI są zielone (helm lint, testy orders-api i payments).
3. Opis PR-a jasno mówi, co się zmienia i dlaczego, oraz podaje zakres, ryzyko i sposób wycofania.

## Blokery
brak

## Uwagi do poprawy
brak

## GitHub Actions
| Check | Stan |
|---|---|
| helm lint | pass |
| test orders-api (Python 3.12, uv) | pass |
| test payments (Go 1.23) | pass |
| build ${{ matrix.service }} | skipping (uruchamiany tylko po push do main) |
| deploy → ${{ vars.K8S_NAMESPACE }} | skipping (uruchamiany tylko po push do main) |

## Uzasadnienie
To kosmetyczna zmiana tekstu stopki. Nie łamie żadnej reguły z REVIEW.md, a CI przechodzi. Ręczny test w przeglądarce, opisany w PR jako planowany, warto wykonać, ale nie blokuje merge.
