# Przegląd PR #2

- **PR:** #2 — orders-api: szybciej
- **Werdykt:** NIE MERGUJ

## 3 najważniejsze powody
1. `permissions: write-all` na poziomie całego workflow `.github/workflows/ci.yml`.
2. Obraz orders-api z tagiem `latest` w `values.yaml`.
3. `actions/checkout` odpięty od pełnego SHA (`@v4`), a do tego usunięte `resources.limits` i opis „drobne poprawki”.

## Blokery
- `.github/workflows/ci.yml:16` — reguła „permissions: write-all w workflow” (zamiast `contents: read`).
- `.github/workflows/ci.yml:39` — reguła „akcja GitHub nieprzypięta do pełnego SHA” (`actions/checkout@v4` zamiast `@3d3c42e…`; przy okazji to downgrade z v7.0.1 do v4).
- `app/deploy/helm/kantyna/values.yaml:78` — reguła „obraz z tagiem latest albo bez tagu” (`tag: latest` zamiast `0.0.0-dev`).

## Uwagi do poprawy
- `app/deploy/helm/kantyna/values.yaml:92-93` (usunięte stare linie 93–95) — reguła „usunięte resources.limits albo requests”: usunięto limity orders-api (`cpu: 500m`, `memory: 384Mi`).
- Opis PR-a — reguła „opis PR-a nie mówi, co i dlaczego się zmienia”: „drobne poprawki” nie opisuje zmian bezpieczeństwa w CI ani zmian w Helm, a tytuł „szybciej” nie ma pokrycia w diffie.
- `.github/workflows/ci.yml:36-37, 57-58, 75-76` — joby testowe i helm lint dostały `contents: read` na poziomie joba, a `build` i `deploy` mają własne bloki `permissions`. Mimo to globalne `permissions: write-all` (linia 16) łamie REVIEW.md: każdy przyszły job bez własnego bloku dostanie pełne uprawnienia tokenu.

## GitHub Actions
| Check | Stan |
|---|---|
| helm lint | pass |
| test orders-api (Python 3.12, uv) | pass |
| test payments (Go 1.23) | pass |
| build ${{ matrix.service }} | skipping |
| deploy → ${{ vars.K8S_NAMESPACE }} | skipping |

## Uzasadnienie
PR spełnia trzy warunki blokujące z REVIEW.md: `write-all`, tag `latest` i akcję nieprzypiętą do SHA. Do tego usuwa limity zasobów pod lakonicznym opisem. Zielone CI tego nie zmienia, bo testy nie sprawdzają tych reguł.
