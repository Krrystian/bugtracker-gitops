# bugtracker-gitops

Repozytorium GitOps dla aplikacji [bugtracker-app](https://github.com/Krrystian/bugtracker-app).
Argo CD wdraża to, co jest na gałęzi **`main`**.

| Gałąź | Rola |
|---|---|
| `development` | automatyczne commity z CI repo aplikacji (nowy tag obrazu) |
| `main` | to, co wdraża Argo CD; zmiany wyłącznie przez ręcznie scalany PR `development` → `main` |

## Struktura

| Ścieżka | Opis |
|---|---|
| `environments/development/values.yaml` | wartości Helm dla środowiska (m.in. `image.tag` = SHA commita) |
| `argocd/application.yaml` | `Application` Argo CD (Multiple Sources: wykres z repo aplikacji + values stąd) |

## Automatyczna aktualizacja tagu (etap 10)

Workflow `.github/workflows/update-image-tag.yml` (na `main`, bo `repository_dispatch` uruchamia tylko domyślna gałąź) reaguje na zdarzenie `image-published` wysyłane przez CI repo aplikacji **wyłącznie z jej `main`**, po opublikowaniu obrazu:

1. waliduje payload (tag = 40-znakowy SHA, źródło = `Krrystian/bugtracker-app`),
2. dociąga `main` do `development`,
3. ustawia `image.tag` w `environments/development/values.yaml` i commituje **tylko na `development`**.

Wdrożenie: ręczny PR `development` → `main` i merge (brama decyzyjna). Webhook → Argo CD wdraża w kilka sekund.

## Wdrożenie Application

```bash
kubectl apply -f argocd/application.yaml
kubectl -n argocd get applications
```

Sekrety nie są przechowywane w tym repozytorium jawnie (od etapu 12 wyłącznie jako `SealedSecret`).
