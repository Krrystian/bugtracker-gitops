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
| `environments/development/secrets/*.sealedsecret.yaml` | sekrety aplikacji jako **SealedSecret** (zaszyfrowane kluczem publicznym kontrolera; odszyfrowuje je wyłącznie kontroler w klastrze) |
| `argocd/application.yaml` | `Application` Argo CD (Multiple Sources: wykres z repo aplikacji + values stąd + katalog `secrets/`) |
| `.gitleaks.toml`, `.github/workflows/gitleaks.yml` | skan sekretów całej historii; jedyny wyjątek: szyfrogram SealedSecret |

## Piaskownica `sandbox` (etap 14, test T3)

`argocd/application-sandbox.yaml` — osobna `Application` wdrażająca gałąź **`vuln/unsigned-image`** do namespace `sandbox`: tylko ręczna synchronizacja, bez Ingress (niedostępna przez Traefik i tunel), bez bazy. Służy wyłącznie do sprawdzenia, że Kyverno odrzuca niepodpisany obraz. Produkcyjna `Application` (`argocd/application.yaml`) śledzi wyłącznie `main`.

```bash
kubectl apply -f argocd/application-sandbox.yaml
argocd app sync bugtracker-sandbox      # lub w UI: SYNC
kubectl -n sandbox get events --field-selector reason=FailedCreate
```

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

## Sekrety (Sealed Secrets, etap 12)

Jawne sekrety **nigdy** nie trafiają do tego repozytorium — tylko `SealedSecret`. Kontroler Sealed Secrets v0.40.0 działa w klastrze (`kube-system`), `kubeseal` 0.40.0 lokalnie.

Nowy lub zmieniony sekret (wartości czytane z klawiatury, nie z historii powłoki):

```bash
kubeseal --fetch-cert > pub-cert.pem                 # klucz publiczny — można go udostępniać
read -rs DB_PASSWORD; read -rs SECRET_KEY
kubectl create secret generic bugtracker-secrets -n bugtracker --dry-run=client -o json \
  --from-literal=db-password="$DB_PASSWORD" --from-literal=secret-key="$SECRET_KEY" \
  | kubeseal --cert pub-cert.pem --format yaml > environments/development/secrets/bugtracker-secrets.sealedsecret.yaml
unset DB_PASSWORD SECRET_KEY
```

**Kopia klucza prywatnego kontrolera** (bez niej po odtworzeniu klastra żadnego `SealedSecret` z tego repo nie da się odszyfrować) — przechowywać **poza repozytorium**, np. w menedżerze haseł:

```bash
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.backup.yaml
```

Odtworzenie: `kubectl apply -f sealed-secrets-key.backup.yaml` przed instalacją kontrolera (lub restart kontrolera po apply).
