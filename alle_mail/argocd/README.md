# Definicje aplikacji ArgoCD

Oba środowiska renderują ten sam chart z `alle_mail/`; różni je wyłącznie plik
`values-<env>.yaml` w `spec.source.helm.valueFiles`.

Przed pierwszym `kubectl apply` potwierdź trzy pola — powstały bez dostępu do klastra:

- **`metadata.name`** musi być identyczna z istniejącą aplikacją (`argocd app list`),
  inaczej ArgoCD utworzy drugą obok zamiast zaktualizować istniejącą.
- **`metadata.namespace`** — `argo` wynika z `argocd-cm.yml` w korzeniu repo.
- **`destination.server`** — UAT działa na węzłach `k102`/`k141`, PROD na `baldur`/`halina512`.
  Jeśli to osobne klastry, `app-uat.yaml` musi wskazywać serwer klastra UAT, a nie
  `https://kubernetes.default.svc`.

Sprawdź też, czy `repoURL` zgadza się z tym, co ArgoCD ma zarejestrowane — tu wpisany jest
adres SSH z `git remote`, ale repozytorium mogło zostać dodane po HTTPS.
