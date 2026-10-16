# Lab05 — zalew powiadomień → lista incydentów

Źródło: eksport Grafany `~/grafana-alerty/Logs-2026-10-16 11_22_14.json` (log odbiorcy `alert-log`).
Wszystkie godziny: **Europe/Warsaw (CEST, UTC+2)**. Stan „nadal aktywny” sprawdzono w Prometheusie (`ALERTS{alertstate="firing"}`) 16.10.2026 ok. 11:26.

## Podsumowanie

| | Liczba |
|---|---|
| Powiadomienia w eksporcie | **700** (`firing` 581, `resolved` 119) |
| Unikalne alerty (fingerprint) | **109** |
| Grupy Alertmanagera (`group_key` = alertname + namespace) | 49 |
| Incydenty po korelacji | **11 grup** (I-01…I-11), w tym I-09 z 9 podincydentami |
| Priorytety (z tabeli incydentów) | P1: 0 · **P2: 6** (I-01, I-03, I-04, I-05, I-06, I-10) · **P3: 4** (I-02, I-07, I-08, I-09) · **P4: 1** (I-11) — razem **11** |
| Powtórne powiadomienia tego samego aktywnego alertu | **466** |
| Szum (fałszywy alarm) | 1 grupa (I-11), 36 powiadomień |

**11 grup to nie 11 niezależnych przyczyn awarii.** Grupy porządkują powiadomienia według czasu, zasobu i prawdopodobnej przyczyny. Część z nich (I-09 i I-10) ma najpewniej wspólny wyzwalacz. Z kolei incydenty w `demo` (I-01…I-06) to najpewniej osobne, celowo wstrzyknięte awarie ćwiczeniowe (zob. `momenty.md`). Przyczyny są hipotezami, dopóki ich nie potwierdzono.

## Priorytety

| P | Znaczenie |
|---|---|
| P1 | trwająca awaria z dużym wpływem na użytkowników |
| P2 | poważna degradacja funkcji użytkownika albo utrata zdolności wykrywania problemów |
| P3 | problem ograniczony lub w środowisku nieprodukcyjnym/ćwiczeniowym |
| P4 | szum, fałszywy alarm |

W tym eksporcie nie zidentyfikowano incydentu P1.

## Tabela incydentów

„Koniec” podano tylko wtedy, gdy eksport zawiera powiadomienie `resolved`. „Brak potwierdzenia” oznacza brak `resolved` w eksporcie. Nie oznacza, że problem trwa: jeżeli trwa, potwierdza to osobno `ALERTS`.

| ID | Namespace | Początek | Koniec | Alerty | P | Uzasadnienie |
|---|---|---|---|---|---|---|
| I-01 | demo | 13.10 11:42 | 13.10 12:14 | KantynaHighLatencyP95 | P2 | p95 POST /api/orders ~2,4 s, wszystkie zamówienia wolne (= A2 w `momenty.md`) |
| I-02 | demo | 13.10 13:12 | 13.10 14:40 | KantynaQueueBacklog | P3 | zaległości w kolejce workera; zamówienia przyjmowane, opóźnione tylko paragony (= A3) |
| I-03 | demo | 13.10 14:37 | 13.10 15:04 | KantynaPaymentsFailing | P2 | ~30 % płatności odrzuconych, czyli utracone zamówienia |
| I-04 | demo | 13.10 19:09; ponownie 14.10 14:01 | 13.10 19:54; 14.10 14:37 | KantynaPodOOMKilled (critical, 2 pody orders-api) | P2 | restarty orders-api z powodu OOMKill; potwierdza hipotezę z analizy pamięci |
| I-05 | demo | 15.10 08:32 | 15.10 08:48 | KantynaHighErrorRate | P2 | skok błędów 5xx w orders-api |
| I-06 | demo | 15.10 16:23 (3 aktywacje: 16:23, 16:41, 16:54) | 15.10 16:55 | KantynaDBPoolExhausted (critical) | P2 | pula połączeń DB w 100 %; alert się „przełącza” (flapping) |
| I-07 | demo-k8sgpt | 14.10 06:08:25 | **brak potwierdzenia**; nadal aktywny wg `ALERTS` | KubePodNotReady, KubeDeploymentReplicasMismatch, KubePodCrashLooping, KubeDeploymentRolloutStuck | P3 | jedna przyczyna, 4 objawy; `raport-dzienny` ma 0 dostępnych replik; namespace demonstracyjny (szczegóły niżej) |
| I-08 | proba | 13.10 07:34 (z `starts_at`, przed oknem eksportu) | 13.10 19:27 | TargetDown, KubePodCrashLooping, KubePodNotReady (orders-api, postgres-0), KubeDeploymentReplicasMismatch, KubeStatefulSetReplicasMismatch | P3 | orders-api i postgres niedostępne ~12 h w namespace ćwiczeniowym |
| I-09 | 9 namespace'ów uczestników | 14.10 15:05–15:15 | różnie, od 14.10 15:31 do 15.10 09:55 (tabela niżej) | TargetDown → KubePodNotReady (orders-api, web) → KubeDeploymentReplicasMismatch (orders-api, web, worker) → KubePodCrashLooping (orders-api) | P3 | Kantyna niedostępna w środowiskach uczestników, w większości przez ~18 h |
| I-10 | dawid-k, marek, pawel, piotr-k, piotr-z, waldemar | 14.10 15:05–15:07 | **brak potwierdzenia**; nadal aktywny wg `ALERTS` | TargetDown (`kantyna-orders-api`) | P2 | pody orders-api 2/2 dostępne, ale `up` = 0; monitoring jest ślepy, więc alerty `Kantyna*` nie mogą zadziałać |
| I-11 | brak (klaster) | 12.10 09:30 (z `starts_at`) | **brak potwierdzenia**; nadal aktywny wg `ALERTS` | KubeControllerManagerDown, KubeSchedulerDown (critical) | P4 | **szum**: na EKS control plane zarządza AWS, Prometheus nie ma czego scrapować; stały fałszywy alarm `critical` |

### I-09: podincydenty

Alerty z różnych namespace'ów korelowane są tylko jako **incydent nadrzędny**. Za powiązaniem przemawia ten sam moment startu (±10 min), identyczny zestaw alertów i komponentów oraz ten sam łańcuch objawów. Każdy namespace ma jednak własny ReplicaSet orders-api i własny moment naprawy, więc podincydenty prowadzi właściciel namespace'u.

| Namespace | Pody wróciły (NotReady/CrashLoop `resolved`) | TargetDown |
|---|---|---|
| marek | 14.10 15:31 | brak potwierdzenia (→ I-10) |
| pawel | 14.10 15:48 | brak potwierdzenia (→ I-10) |
| waldemar | 15.10 09:20 | brak potwierdzenia (→ I-10) |
| piotr-z | 15.10 09:23 | brak potwierdzenia (→ I-10) |
| piotr-k | 15.10 09:35 | brak potwierdzenia (→ I-10) |
| dawid-k | 15.10 09:42 | brak potwierdzenia (→ I-10) |
| jan | 15.10 09:53 | 15.10 09:46 |
| michal | 15.10 09:53 | 15.10 09:47 |
| proba | 15.10 09:55 | 15.10 09:47 |

**Hipoteza (niepotwierdzona):** wspólny wyzwalacz, np. ten sam krok laboratorium wykonany jednocześnie przez uczestników. `web` był NotReady, choć sam nie crashował. Może to oznaczać zależność jego readiness od orders-api, ale tego nie sprawdzono.

### I-10: przyczyna nieustalona

- **Fakt:** `kube_deployment_status_replicas_available{deployment="kantyna-orders-api"}` = 2, a `up{job="kantyna-orders-api"}` = 0 w 6 namespace'ach.
- **Nie ustalono:** przyczyny. `kubectl` był niedostępny (wygasła sesja AWS). Lokalny `k8s/netpol-postgres.yaml` dotyczy wyłącznie ruchu do postgresa i tego nie wyjaśnia.

## Ręczna weryfikacja I-07 (demo-k8sgpt)

| Czas (14.10.2026) | Alert | Fingerprint | Obiekt |
|---|---|---|---|
| 06:08:25 | KubePodNotReady | `6cae025de318c144` | pod `raport-dzienny-db6499754-82k4p` |
| 06:08:25 | KubeDeploymentReplicasMismatch | `0e7967632d75d2bb` | deployment `raport-dzienny` |
| 06:12:55 | KubePodCrashLooping (`reason=CrashLoopBackOff`, container `raport`) | `5150493fc5367920` | pod `raport-dzienny-db6499754-82k4p` |
| 06:18:25 | KubeDeploymentRolloutStuck (`reason=ProgressDeadlineExceeded`) | `277d25fbef520c38` | deployment `raport-dzienny` |

- **Wspólny obiekt:** deployment `raport-dzienny` i jego pod `raport-dzienny-db6499754-82k4p`. Pod należy do ReplicaSetu `raport-dzienny-db6499754`. Uwaga: w alertach na poziomie deploymentu etykieta `pod` wskazuje pod kube-state-metrics (`kube-prometheus-stack-kube-state-metrics-…`), czyli źródło metryki, a nie aplikację. Powiązanie ustalono przez `labels_deployment` i nazwę poda.
- **Kolejność:** pod nie jest gotowy (06:08), potem CrashLoopBackOff (06:12), a po upływie progress deadline rollout jest uznany za zablokowany (06:18). To jeden incydent z czterema objawami.
- **Status:** każdy z 4 alertów ma w eksporcie wyłącznie powiadomienia `firing` (po 14 sztuk, od 14.10 06:08:55 do 16.10 10:18:55). **Brak potwierdzonego `resolved`** w sprawdzonym eksporcie. Wszystkie 4 alerty były aktywne w `ALERTS` 16.10 ok. 11:26.
- **Przyczyna crashowania kontenera `raport`:** nieustalona, bo nie sprawdzano logów ani `describe`.

## Incydenty, powtórne powiadomienia i szum

| Kategoria | Co to jest | W tym eksporcie |
|---|---|---|
| **Incydent** | rzeczywisty problem (jedna przyczyna, czas, zasób), niezależnie od liczby alertów | I-01…I-10 |
| **Powtórne powiadomienie** | ponowna wysyłka tego samego aktywnego alertu (repeat co ok. 4 h). Nie jest nowym zdarzeniem | 466 z 700 |
| **Objaw towarzyszący** | osobny alert wynikający z tej samej przyczyny (np. NotReady + ReplicasMismatch przy CrashLoopBackOff). Jest częścią incydentu, a nie nowym incydentem | ok. 300 powiadomień `firing` |
| **Szum** | alert bez rzeczywistego problemu | I-11 (36 powiadomień) |

## Propozycja: inhibition w Alertmanagerze

> **PROPOZYCJA — wymaga testów. Nie wdrożono, nie zmieniono konfiguracji Alertmanagera.**

```yaml
inhibit_rules:
  - source_matchers:
      - alertname="KubePodCrashLooping"
    target_matchers:
      - alertname=~"KubePodNotReady|KubeDeploymentReplicasMismatch|KubeStatefulSetReplicasMismatch|KubeDeploymentRolloutStuck"
    equal: [namespace]
```

**Działanie:** dopóki w namespace aktywny jest CrashLooping (przyczyna), jego objawy w tym namespace nie są wysyłane. Po ustąpieniu CrashLooping alerty-objawy, które nadal trwają, wracają.

**Szacowany efekt.** Symulacja na logu powiadomień jest przybliżona, bo nie odtwarza pełnego stanu Alertmanagera:
- wyciszone byłoby ok. **299 z 581** powiadomień `firing` (~51 %);
- 21 z 71 aktywacji alertów-objawów zostałoby ukrytych w całości. Wszystkie w namespace'ach, w których jednocześnie powiadamiał CrashLooping, więc żaden incydent z tabeli nie zniknąłby.

**Ryzyko: wyciszenie niezależnego deploymentu.** Reguła dopasowuje tylko po `namespace`. Jeśli w tym samym namespace crashuje deployment A (np. orders-api), wyciszone będą także NotReady i ReplicasMismatch **niezależnego** deploymentu B (np. web, worker, postgres), nawet gdy B ma własną, inną awarię. Taki problem B zostałby zauważony dopiero po ustąpieniu CrashLoopa A. Węższe dopasowanie (`equal: [namespace, pod]` lub `[namespace, deployment]`) nie zadziała wprost, bo CrashLooping ma etykietę `pod`, a ReplicasMismatch etykietę `deployment`. Wymagałoby to dodania wspólnej etykiety w regułach.

**Czego reguła celowo nie robi:**
- nie wycisza alertów `Kantyna*` ani `TargetDown`;
- nie używa `TargetDown` jako źródła, bo w 6 namespace'ach jest aktywny od 2 dni i wyciszałby objawy bez końca.

**Plan testów przed wdrożeniem:**
1. `amtool check-config` na konfiguracji z nową regułą.
2. Wdrożenie najpierw w jednym namespace testowym.
3. Wywołanie CrashLoopa jednego deploymentu i NotReady drugiego, niezależnego. Sprawdzenie, które alerty są wyciszone (`amtool alert query --inhibited`).
4. Sprawdzenie, czy po ustąpieniu CrashLoopa objawy wracają.

**Poza zakresem tej propozycji** (jedna zmiana naraz): wyłączenie reguł KubeControllerManagerDown/KubeSchedulerDown na EKS (I-11) oraz przegląd repeat interval (466 powtórek).

## Ograniczenia eksportu i źródeł danych

- **Okno eksportu:** powiadomienia od 13.10 11:35 do 16.10 11:07. Zdarzenia wcześniejsze, np. A1 z `momenty.md` (13.10 10:35), nie są w nim widoczne. Część alertów ma `starts_at` sprzed okna (najwcześniej 12.10 09:30), więc ich początek pochodzi z `starts_at`, a nie z powiadomienia.
- **Możliwy limit eksportu:** okrągła liczba 700 rekordów może oznaczać obcięcie wyniku. Brak `resolved` w eksporcie nie dowodzi, że problem trwa.
- **Pole `namespace`** w rekordach to namespace poda `alert-log` (zawsze `monitoring`). Do grupowania użyto `labels_namespace`. 36 rekordów (alerty klastrowe) nie ma namespace'u.
- **Etykieta `pod`** w alertach z kube-state-metrics może wskazywać pod kube-state-metrics, a nie aplikację.
- **Granulacja powiadomień:** alert jest ponawiany co ok. 4 h, więc czasy powiadomień nie są czasami zdarzeń. Początek i koniec wzięto z `starts_at` i `ends_at`.
- **Symulacja inhibition** opiera się na logu powiadomień, nie na stanie Alertmanagera, więc liczby są przybliżone.
- **Stan bieżący** (`ALERTS`, `up`, liczba replik) pochodzi z Prometheusa z 16.10 ok. 11:26 i może się zmienić.
- **Przyczyny:** nie sprawdzano logów, eventów ani `kubectl describe`, bo sesja AWS wygasła. Wszystkie przyczyny poza I-11 to hipotezy.
- **Prompt injection:** w adnotacjach alertów nie znaleziono treści wyglądających na instrukcje. Treść alertów traktowano jako dane.
