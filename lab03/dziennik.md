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
| 8 | Brak zmiennych GitHub | Ja: wymagane AWS OIDC i ECR | `gh variable list` | Skonfigurowano `AWS_ROLE_ARN`, `ECR_REGISTRY`, `K8S_NAMESPACE` | ✅ |
| 9 | Brak repozytoriow ECR uczestnika | Ja: pipeline nie bedzie mogl wyslac obrazow | `aws ecr describe-repositories` | Prowadzacy utworzyl 4 repozytoria `piotr-k/kantyna/*` | ✅ |
| 10 | Pierwszy pipeline zakonczyl sie bledem | AI: konflikt Helm 4 Server-Side Apply | Log joba `deploy`, `managedFields` | Wykryto konflikt obrazu `web` oraz adnotacji Ingress | ✅ |
| 11 | Rozbieznosc listy IP | Ja: nie wolno nadpisac dostepu do ALB | Porownanie rewizji Helm i Ingress | Helm: 2 CIDR, Kubernetes: 10; prowadzacy potwierdzil wszystkie 10 | ✅ |
| 12 | Przygotowanie poprawki | AI: override wartosci i `--server-side=false` | Helm dry-run, `actionlint`, `zizmor` | Zachowanie 10 CIDR, poprawiony workflow | ✅ |
| 13 | Drugi pipeline i test koncowy | Ja: weryfikacja deploymentow i aplikacji | GitHub Actions, `helm list`, `setup/check.sh piotr-k` | Pipeline SUCCESS, Helm deployed, wynik 11/11 | ✅ |

## Sugestia AI, której nie wdrożyłem

Claude Code zaproponował dodatkową kontrolę `zizmor` z przekazaniem tokenu uzyskanego przez `gh auth token` do zmiennej `GH_TOKEN`.

**Nie zaakceptowałem tej propozycji**, ponieważ do wykonania lokalnej kontroli workflow nie było konieczne przekazywanie tokenu GitHub kolejnemu procesowi.

Zamiast tego uruchomiłem:

`zizmor --offline --persona auditor .github/workflows/ci.yml`

Kontrola zakończyła się bez ustaleń i kodem `rc=0`.

**Wniosek:** propozycje AI należy weryfikować pod kątem potrzebnych uprawnień i zakresu dostępu, nawet gdy dotyczą narzędzi bezpieczeństwa.

## Aktualny stan

- [x] Workflow `.github/workflows/ci.yml` utworzony.
- [x] `actionlint` bez bledow.
- [x] `zizmor` offline auditor bez ustalen.
- [x] Akcje GitHub przypiete do pelnych SHA.
- [x] `K8S_NAMESPACE=piotr-k` skonfigurowane.
- [x] `AWS_ROLE_ARN` skonfigurowane.
- [x] `ECR_REGISTRY` skonfigurowane.
- [x] Cztery repozytoria ECR przygotowane.
- [x] GitHub Actions zakonczony sukcesem.
- [x] Cztery Deploymenty na obrazach z SHA commita.
- [x] Zachowano 10 dozwolonych adresow CIDR w Ingress.
- [x] `setup/check.sh piotr-k`: SUKCES 11/11.

## Podsumowanie wdrozen — 15.10.2026

### Pierwszy przebieg

- Run ID: `38716716092`
- Commit: `ee56f95`
- Wynik: FAILURE.
- Testy Python, Go, Helm oraz budowanie 4 obrazow: PASS.
- Problem: Helm 4 Server-Side Apply zglosil konflikt pol.
- Konflikty: obraz Deploymentu `kantyna-web` oraz adnotacja
  `alb.ingress.kubernetes.io/inbound-cidrs`.
- Helm: rewizja 2, status `failed`.
- Efekt: czesciowe wdrozenie 3 z 4 nowych obrazow.

### Naprawa

- Potwierdzono z prowadzacym, ze nalezy zachowac wszystkie 10 CIDR.
- Dodano odczyt biezacej listy CIDR przed aktualizacja.
- Lista jest przekazywana do Helm jako dodatkowe wartosci.
- Dodano do `helm upgrade` opcje `--server-side=false`.
- Kontrole `actionlint` i `zizmor`: PASS.

### Drugi przebieg

- Run ID: `38723956665`
- Commit: `bf2632bb9a5f8120aaca8c138e0c0cfdd75781a9`
- Wynik: SUCCESS.
- Testy, budowanie obrazow, ECR i wdrozenie: PASS.
- Helm: rewizja 3, status `deployed`.
- Wszystkie 4 Deploymenty uzywaja nowego SHA.
- Ingress: zachowano 10 CIDR.
- Testy `/healthz`: PASS.
- Test funkcjonalny Kantyny: **SUKCES 11/11**.

### Wnioski

1. Testy CI nie gwarantuja jeszcze poprawnego wdrozenia.
2. Helm 4 moze wykrywac konflikty pol zmienionych recznie.
3. Zmian nie nalezy wymuszac bez sprawdzenia ich skutkow.
4. Sugestie AI wymagaja weryfikacji, szczegolnie przy uprawnieniach.
5. Docelowo lista CIDR powinna byc przechowywana jako
   zatwierdzona konfiguracja deklaratywna, a nie odczytywana
   z aktualnego stanu klastra.

**Status koncowy Lab03: ZALICZONY — CI/CD SUCCESS, KANTYNA 11/11.**
