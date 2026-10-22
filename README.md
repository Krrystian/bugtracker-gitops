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

## Wdrożenie Application

```bash
kubectl apply -f argocd/application.yaml
kubectl -n argocd get applications
```

Sekrety nie są przechowywane w tym repozytorium jawnie (od etapu 12 wyłącznie jako `SealedSecret`).
