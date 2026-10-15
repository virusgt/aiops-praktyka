# Przegląd PR

- **PR:** #2 — orders-api: szybciej
- **Werdykt:** NIE MERGUJ

## 3 najważniejsze powody
1. Workflow dostaje `permissions: write-all` na poziomie globalnym (`.github/workflows/ci.yml:16`). Wcześniej było `contents: read`.
2. `actions/checkout` jest odpięty z pełnego SHA na ruchomy tag `@v4` (`.github/workflows/ci.yml:39`).
3. Obraz orders-api dostaje tag `latest` (`app/deploy/helm/kantyna/values.yaml:78`), a jego `resources.limits` zostały usunięte.

## Blokery
- `.github/workflows/ci.yml:16`: `permissions: write-all` (reguła: *permissions: write-all w workflow*).
- `.github/workflows/ci.yml:39`: `uses: actions/checkout@v4` zamiast pełnego SHA (reguła: *akcja GitHub nieprzypięta do pełnego SHA*).
- `app/deploy/helm/kantyna/values.yaml:78`: `ordersApi.image.tag: latest` (reguła: *obraz z tagiem latest albo bez tagu*).

## Uwagi do poprawy
- `app/deploy/helm/kantyna/values.yaml:93` (w starym pliku linie 93–95): usunięte `ordersApi.resources.limits` (`cpu: 500m`, `memory: 384Mi`). Reguła: *usunięte resources.limits albo requests*.
- Opis PR-a to tylko „drobne poprawki”. Nie mówi, co się zmienia ani dlaczego, i nie uzasadnia tytułu „szybciej”. Reguła: *opis PR-a nie mówi, co i dlaczego się zmienia*.
- Do jobów test orders-api, test payments i helm lint dodano `permissions: contents: read` (`ci.yml:36-37`, `:57-58`, `:75-76`). Joby `build` i `deploy` mają własne bloki `permissions` (`contents: read`, `id-token: write`; na `main` w `ci.yml:91-93` i `:128-130`), których PR nie zmienia. Dziś żaden job nie dziedziczy więc `write-all`. Globalne `write-all` zostaje jednak domyślnym zakresem dla każdego nowego joba bez własnego bloku i i tak łamie regułę z REVIEW.md.

## GitHub Actions
| Check | Stan |
|---|---|
| helm lint | pass (5s) |
| test orders-api (Python 3.12, uv) | pass (12s) |
| test payments (Go 1.23) | pass (17s) |
| build ${{ matrix.service }} | skipping |
| deploy → ${{ vars.K8S_NAMESPACE }} | skipping |

Zielone checki nie pokrywają problemów z tego PR-a. Lint i testy nie sprawdzają uprawnień workflow, przypięcia akcji, tagów obrazów ani limitów zasobów.

## Uzasadnienie
PR łamie trzy reguły blokujące z REVIEW.md: `write-all` w workflow, akcja nieprzypięta do SHA i obraz `latest`. Dochodzą do tego usunięte limity i pusty opis. Do merge potrzebne są przywrócone `contents: read`, SHA akcji, wersjonowany tag obrazu i `limits`, oraz opis zmiany.
