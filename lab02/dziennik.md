# Lab02 — Dziennik diagnozy Kantyny

- **Środowisko:** AWS EKS `aiops`, namespace `piotr-k`
- **Narzędzia:** kubectl, Claude Code, k8sgpt, LiteLLM
- **Wynik początkowy:** 5/11
**Wynik końcowy:** SUKCES 11/11

## Dziennik napraw

| # | Objaw | Hipoteza i dowód | Komenda weryfikująca | Wynik | Status | Poprawka |
|---|---|---|---|---|---|---|
| 1 | Web: ImagePullBackOff | Ręcznie ustawiono obraz `web:1.0.1`; rejestr zwracał NotFound. Poprzedni ReplicaSet wskazywał `1.0.0`. | `kubectl -n piotr-k get rs -l app.kubernetes.io/name=web` | Web 2/2, wynik 6/11 | ✅ | Rollback Deploymentu do rewizji 1 |
| 2 | Worker: FailedCreate | Request pamięci 2Gi przekraczał dostępne 1600Mi w ResourceQuota. Poprzednia konfiguracja miała 128Mi/256Mi. | `kubectl -n piotr-k describe resourcequota participant-quota` | Worker 1/1, wynik 7/11 | ✅ | Rollback Deploymentu do rewizji 1 |
| 3 | Payments: brak endpointów | Service wybierał `payment`, a pody miały etykietę `payments`. | `kubectl -n piotr-k get svc kantyna-payments -o yaml` | Dwa endpointy, wynik 8/11 | ✅ | Poprawa selektora Service |
| 4 | Orders API: CrashLoopBackOff | DATABASE_URL wskazywał nieistniejący host `kantyna-postgresql` zamiast `kantyna-postgres`. | `kubectl -n piotr-k get svc kantyna-postgresql` | API 2/2, wynik 9/11 | ✅ | Rollback Deploymentu do rewizji 1 |
| 5 | Menu i zamówienia: timeout | NetworkPolicy dopuszczała ruch z `frontend`, podczas gdy pody web miały etykietę `web`. | `kubectl -n piotr-k describe networkpolicy kantyna-orders-api-ingress` | HTTP 200; wynik 11/11 | ✅ | Poprawa selektora nadawcy w NetworkPolicy |

## Porównanie k8sgpt i Claude Code

k8sgpt wykrył problemy z pobieraniem obrazu, przekroczeniem ResourceQuota i brakiem endpointów Service.

Claude Code pozwolił dodatkowo przeanalizować historię Deploymentów, ReplicaSety, konfigurację chartu Helm i przyczyny zmian.

Ostateczne przyczyny potwierdzano niezależnymi poleceniami kubectl przed wykonaniem poprawek.

## Sugestie AI, których nie wdrożyłem

1. Usunięcie `kube-root-ca.crt` — brak wykorzystania ConfigMap nie jest wystarczającym powodem do jej usunięcia.
2. Instalacja AWS Load Balancer Controller — nie było potwierdzenia, że wymaga tego nasza awaria; to również zmiana o szerszym zasięgu.
3. Zwiększenie ResourceQuota — należało przywrócić właściwy request pamięci workera.
4. Restartowanie podów bez ustalenia przyczyny — nie rozwiązałoby niedostępnego obrazu, błędnego DNS ani niewłaściwej NetworkPolicy.

## Ograniczenia diagnozy

Bezpośrednia weryfikacja tagów ECR była niemożliwa z powodu braku uprawnienia `ecr:DescribeImages`. Wykorzystano zdarzenia Kubernetes, historię ReplicaSetów i wyniki działania podów.

## Wniosek

Najpierw zbieram dowody, następnie weryfikuję hipotezy AI, wykonuję jedną uzasadnioną zmianę i ponawiam testy.

**Wynik końcowy: SUKCES 11/11.**