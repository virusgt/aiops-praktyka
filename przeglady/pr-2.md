# Przegląd PR #2

- **PR:** #2 — orders-api: szybciej
- **Werdykt:** NIE MERGUJ

## 3 najważniejsze powody
1. Globalne `permissions: write-all` (`.github/workflows/ci.yml:16`) zastępuje dotychczasowe `contents: read`.
2. Obraz orders-api z tagiem `latest` (`app/deploy/helm/kantyna/values.yaml:78`) daje niepowtarzalne wdrożenia i uniemożliwia wiarygodny rollback.
3. `actions/checkout` jest odpięty z pełnego SHA na ruchomy tag `@v4` (`.github/workflows/ci.yml:39`), co otwiera ryzyko w łańcuchu dostaw.

## Blokery
- `.github/workflows/ci.yml:16`: `permissions: write-all`. Reguła: „permissions: write-all w workflow”. Wszystkie joby mają własne bloki `permissions`: test i lint dostały w tym PR `contents: read`, a `build` i `deploy` mają już `contents: read` + `id-token: write`, którego PR nie zmienia. Żaden job nie dziedziczy więc `write-all`. Mimo to samo globalne `write-all` łamie regułę, a każdy nowy job bez własnego bloku dostanie pełne uprawnienia zapisu.
- `.github/workflows/ci.yml:39`: `uses: actions/checkout@v4`. Reguła: „akcja GitHub nieprzypięta do pełnego SHA”.
- `app/deploy/helm/kantyna/values.yaml:78`: `tag: latest`. Reguła: „obraz z tagiem latest albo bez tagu”.

## Uwagi do poprawy
- `app/deploy/helm/kantyna/values.yaml:93` (usunięte linie 93–95 starej wersji): usunięte `resources.limits` (cpu 500m, memory 384Mi) dla orders-api. Reguła: „usunięte resources.limits albo requests”.
- Opis PR-a („drobne poprawki”). Reguła: „opis PR-a nie mówi, co i dlaczego się zmienia”. Tytuł „szybciej” nie ma pokrycia w zmianach: żadna z nich nie przyspiesza aplikacji.

## GitHub Actions
| Check | Stan |
|---|---|
| helm lint | pass |
| test orders-api (Python 3.12, uv) | pass |
| test payments (Go 1.23) | pass |
| build ${{ matrix.service }} | skipping (uruchamiany tylko po push do main) |
| deploy → ${{ vars.K8S_NAMESPACE }} | skipping (uruchamiany tylko po push do main) |

## Uzasadnienie
PR łamie trzy reguły blokujące (write-all, akcja bez SHA, tag latest) i dodatkowo zdejmuje limity zasobów, a jego opis nic nie wyjaśnia. Zielone CI nie zmienia werdyktu, bo checki nie weryfikują tych reguł.
