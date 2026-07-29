# Ujednolicenie UAT i PROD — skrót zmian

Data: 2026-07-29 · gałąź: `unify-uat-prod` · **niezacommitowane**

## Co było

`alle_mail/UAT/` i `alle_mail/PROD/` to były dwie pełne kopie tego samego chartu. Dziewięć z
piętnastu szablonów identycznych bajt w bajt, reszta różniła się o ~27 linii — w większości
przypadkowy dryf, nie realna różnica środowiskowa. Każda zmiana wymagała edycji w dwóch miejscach,
a gdy o tym zapomniano, środowiska się rozjeżdżały (por. commit `0aeeb17`, który usuwał
`05-repeater` z PROD przez skasowanie plików zamiast przełącznika).

## Co jest

```
alle_mail/
├── Chart.yaml
├── values.yaml          # wspólna baza
├── values-uat.yaml      # 4 nadpisania
├── values-prod.yaml     # 8 nadpisań
├── templates/           # JEDNA kopia (19 plików)
├── argocd/              # app-uat.yaml, app-prod.yaml + README
└── k8s_deployment/      # przeniesione z PROD/ (gitignored)
```

Środowiska pozostają w ArgoCD osobnymi aplikacjami (`alle-mail-uat`, `alle-mail-prod`) —
ten sam `path: alle_mail`, różny `helm.valueFiles`. Osobny sync, osobne namespace'y, pełna
rozróżnialność w UI.

## Różnice środowiskowe — teraz wszystkie w jednym miejscu

| | UAT | PROD |
|---|---|---|
| `namespace` | `alle-uat` | `alle-mail` |
| `vault.mount_point` | `alle_dev` | `alle` |
| `image_B.replicas` / `image_D.replicas` | 1 / 1 | 3 / 3 |
| `image_D.external_port` | 40054 | 40055 |
| `image_F.schedule` / `image_G.schedule` | co 3h / co 2h | `*/47` / `*/56` |
| `nodes.tracker` (cronJob F, G) | brak | `baldur` |
| `nodes.traefik` (deployment_D) | brak | `halina512` |
| `ingress.annotations` | tylko nginx rewrite | + `ingress.class: traefik` |
| `image_E.enabled` (05-repeater) | `true` | `false` |
| `image.tag` | ustawiany przez `deploy_to_uat.sh` | przez `deploy_prod.sh` |

## Mechanizmy w szablonach

- **nodeSelector** — `{{- with .Values.nodes.X }}` w `cronJob_F/G.yml` i `deployment_D.yml`.
  `null` = brak `nodeSelector`, więc UAT nie dostaje żadnego.
- **05-repeater** — `deployment_E.yml` i `config_05.yml` owinięte
  `{{- if .Values.image_E.enabled }}`. Włączenie w PROD to jedna linia w values.
- **ingress** — annotacje z `toYaml .Values.ingress.annotations`; PROD dokłada swoją do bazowej
  przez głębokie scalanie map.
- Nie dodano helpera do `_helpers.tpl` — plik to nieużywany boilerplate z `helm create`, żaden
  szablon go nie woła; `with` jest spójne ze stylem reszty chartu.

## Weryfikacja (wykonana)

Porównanie renderów przed/po przez parser YAML, obiekt po obiekcie:

- **UAT — 18 obiektów, wszystkie semantycznie identyczne.** Zero zmian.
- **PROD — 16 obiektów, dokładnie 5 zamierzonych zmian i nic poza nimi:**
  - `revisionHistoryLimit: 2` w czterech Deploymentach (wcześniej domyślne 10)
  - `deimos-config` → `NAMESPACE: "alle-mail"` (było `alle-local`)

Różnice czysto tekstowe (cudzysłowy przy `halina512`, kolejność annotacji po `toYaml`, białe znaki,
usunięty martwy zakomentowany kod) zniknęły przy porównaniu semantycznym — YAML jest równoważny.

`helm lint` przechodzi dla obu środowisk. `05-repeater` i `config-05` renderują się tylko w UAT.
`sed` ze skryptów deploy trafia dokładnie jedną linię w każdym pliku values (sprawdzone symulacją).

## Zmiany poza tym repo

Kanoniczne skrypty w `/net/nas/fast/loper/Projekty/Integromat/Allegro_mail/` (kopie w
`GitLab/alle_mail/` są nadpisywane przez `rsync`, więc ich nie ruszano):

- `deploy_to_uat.sh:11` → `UAT_VALUES="${ARGO_DIR}/values-uat.yaml"`
- `deploy_prod.sh:11` → `PROD_VALUES="${ARGO_DIR}/values-prod.yaml"`

## Zanim zsynchronizujesz — do rozstrzygnięcia

1. **`NAMESPACE` w `deimos-config` — rozstrzygnięte, nie blokuje synchronizacji.**
   Sprawdzone przez grep całego repo: nic w `templates/` nie odwołuje się do
   `deimos-config` przez `configMapRef` — jedyne miejsca, które to robią, to
   `k8s_deployment/deiployer.yaml` i `k8s_deployment/debug.yaml`, oba działające w
   namespace `alle-local`. `configMapRef` rozwiązuje się wyłącznie w tym samym
   namespace co Pod, więc ConfigMap renderowany przez ten chart (do `alle-mail`/
   `alle-uat`) jest **innym obiektem** niż ten, który czyta deployer — Helm/ArgoCD nie
   ma żadnego wpływu na `alle-local`. Cokolwiek tam realnie istnieje, jest utrzymywane
   ręcznie, poza tym chartem; decyzja, czy zaktualizować tamten obiekt, jest osobna i
   nie blokuje tego syncu. Opcjonalne, ręczne potwierdzenie (nie wymagane przed
   syncem):
   ```bash
   kubectl -n alle-local get cm deimos-config -o yaml
   kubectl -n alle-mail  get cm deimos-config -o yaml
   ```
2. **Pola w `argocd/*.yaml`** — częściowo rozstrzygnięte:
   - `repoURL` (`git@github.com:loper/argocd.git`) zgadza się z `git remote -v` tego
     repo — bez rozbieżności.
   - `destination.server: https://kubernetes.default.svc` jest poprawne dla obu
     środowisk — to jeden klaster k3s, separacja UAT/PROD idzie przez namespace +
     nodeSelector (`baldur`/`halina512` dla PROD), nie przez osobne API servery;
     nigdzie w historii tego projektu nie ma śladu wielu klastrów.
   - **Wciąż wymaga ręcznego sprawdzenia**: `metadata.name`
     (`alle-mail-uat`/`alle-mail-prod`) musi zgadzać się z tym, co ArgoCD już ma
     zarejestrowane — `argocd app list` przed pierwszym `kubectl apply`. Nie dało się
     tego zweryfikować z tej sesji: `kubectl` nie miał poświadczeń (ten sam brak
     dostępu, co przy innych punktach w `HANDOFF.md` tego repo-siostrzanego
     Allegro_mail). Zła nazwa utworzy drugą aplikację zamiast zaktualizować istniejącą
     — to jedyny punkt z tej listy, który zostaje twardym warunkiem przed `apply`.
3. **Kolejność.** Najpierw UAT (`argocd app diff alle-mail-uat` → sync), sprawdź działanie,
   dopiero potem PROD.

## Świadomie poza zakresem

- `templates/service_D.yml:9` ma literówkę `- ports:` zamiast `- name:` — renderuje pole `ports`
  o wartości null w elemencie listy portów. Występuje tak samo przed i po, do naprawy osobno.
- Migracja annotacji `kubernetes.io/ingress.class` (deprecated) na `spec.ingressClassName`.
- `helm template` bez `-f values-<env>.yaml` renderuje pusty tag i namespace. ArgoCD i skrypty
  deploy zawsze podają plik środowiskowy, więc nie dodawano `required`.
