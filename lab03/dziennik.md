# Lab03 — Pipeline CI/CD Kantyny z AI

- **Repozytorium:** `virusgt/aiops-praktyka`
- **Namespace:** `piotr-k`
- **Klaster:** AWS EKS `aiops` (`eu-central-1`)
**Narzędzia:** Claude Code, GitHub Actions, actionlint 1.7.12, zizmor 1.31.0, Helm

## Cel laboratorium

Przygotowanie bezpiecznego pipeline'u GitHub Actions, który:

- Uruchamia testy Python i Go.
- Buduje cztery obrazy: `orders-api`, `payments`, `worker`, `web`.
- Publikuje obrazy w AWS ECR z tagiem SHA commita.
- Uwierzytelnia się w AWS przez OIDC, bez kluczy dostępowych.
- Wdraża aplikację do namespace `piotr-k` za pomocą Helm.
- Sprawdza rollout i endpointy `/healthz`.

## Dziennik diagnozy

| # | Objaw (krok, komunikat) | Hipoteza (AI / ja) | Jak sprawdziłem | Poprawka | Status |
|---|---|---|---|---|---|
| 1 | Brak narzędzi `gh`, `actionlint`, `zizmor` | Ja: środowisko WSL wymaga przygotowania | `command -v`, `gh auth status`, sprawdzenie wersji | Instalacja GitHub CLI, actionlint i zizmor | ✅ |
| 2 | `make build` obejmuje także `loadgen` | Ja + AI: standardowy target buduje za dużo obrazów | Analiza `app/Makefile` | Macierz buildów ograniczona do czterech serwisów | ✅ |
| 3 | Potrzebny workflow CI/CD | AI: GitHub Actions z testami, Docker Buildx, AWS OIDC i Helm | Analiza `Makefile`, Dockerfile, chartu Helm i wygenerowanego `ci.yml` | Utworzenie `.github/workflows/ci.yml` | ✅ |
| 4 | `actionlint` — kontrola składni | Ja: workflow może zawierać błędy YAML lub GitHub Actions | `actionlint .github/workflows/ci.yml` | Brak wymaganych poprawek; wynik `rc=0` | ✅ |
| 5 | `zizmor`: 2 pominięte ustalenia | Ja: standardowa kontrola może nie pokazywać wszystkich uwag | `zizmor --offline --persona auditor .github/workflows/ci.yml` | Wykryto dwie uwagi `undocumented-permissions` | ✅ |
| 6 | Brak komentarzy przy `id-token: write` | Ja: należy uzasadnić uprawnienia AWS OIDC | Wynik zizmor dla jobów `build` i `deploy` | Dodanie komentarzy wyjaśniających użycie OIDC | ✅ |
| 7 | Ponowna kontrola bezpieczeństwa | Ja: po poprawce narzędzia powinny przejść bez ustaleń | `actionlint` i `zizmor --offline --persona auditor` | Oba programy zwróciły `rc=0`; `No findings to report` | ✅ |
| 8 | Brak `AWS_ROLE_ARN` oraz `ECR_REGISTRY` | Ja: nie można jeszcze zweryfikować logowania OIDC i wdrożenia | `gh variable list --repo virusgt/aiops-praktyka` | Oczekiwanie na wartości od prowadzącego | ⏳ |

## Sugestia AI, której nie wdrożyłem

Claude Code zaproponował dodatkową kontrolę `zizmor` z przekazaniem tokenu uzyskanego przez `gh auth token` do zmiennej `GH_TOKEN`.

**Nie zaakceptowałem tej propozycji**, ponieważ do wykonania lokalnej kontroli workflow nie było konieczne przekazywanie tokenu GitHub kolejnemu procesowi.

Zamiast tego uruchomiłem:

`zizmor --offline --persona auditor .github/workflows/ci.yml`

Kontrola zakończyła się bez ustaleń i kodem `rc=0`.

**Wniosek:** propozycje AI należy weryfikować pod kątem potrzebnych uprawnień i zakresu dostępu, nawet gdy dotyczą narzędzi bezpieczeństwa.

## Aktualny stan

- [x] Workflow `.github/workflows/ci.yml` utworzony.
- [x] `actionlint` bez błędów.
- [x] `zizmor` w trybie offline auditor bez ustaleń.
- [x] Akcje przypięte do pełnych SHA, sprawdzonych względem tagów.
- [x] `K8S_NAMESPACE=piotr-k` skonfigurowane.
- [ ] `AWS_ROLE_ARN` skonfigurowane.
- [ ] `ECR_REGISTRY` skonfigurowane.
- [ ] Pierwszy przebieg GitHub Actions zakończony sukcesem.
- [ ] Cztery Deploymenty działają na obrazach oznaczonych SHA commita.
- [ ] `setup/check.sh piotr-k` potwierdza `SUKCES: 11/11` po wdrożeniu.

**Status:** lokalne przygotowanie pipeline'u zakończone, wdrożenie oczekuje na konfigurację AWS OIDC.