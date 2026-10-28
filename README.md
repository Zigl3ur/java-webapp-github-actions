# Webapp Java

Application de démo Java (JSP + Servlet).

## Structure du projet

- `server` — logique métier (module `jar`), ex. `GreetingService`.
- `webapp` — interface web JSP (module `war`), dépend de `server`.

## Prérequis

- JDK 11
- Maven 3.9+
- Docker

## Build

```bash
mvn clean package
```

Génère le fichier `webapp/target/webapp.war`.

## Tests

```bash
mvn test
```

## Deploy avec Docker

```bash
docker build -t webapp-java .
docker run -p 8081:8080 webapp-java
```

## CI/CD avec GitHub Actions

Le workflow [`.github/workflows/ci.yml`](.github/workflows/ci.yml) reprend les
cinq étapes de la CI GitLab :

1. `build` : compile avec Maven 3.9.4 et Java 21, sans exécuter les tests,
   puis conserve `webapp/target/webapp.war` comme artefact.
2. `unit-test` : exécute `mvn test` après la compilation.
3. `scan-image` : récupère le WAR, construit l'image Docker et lance Trivy
   avec son cache `.trivycache`. Une vulnérabilité `CRITICAL` bloque la suite,
   même si aucun correctif n'est disponible.
4. `push-image` : publie l'image validée dans GitHub Container Registry (GHCR),
   avec les tags du SHA court (8 caractères) et `latest`.
5. `update-manifest` : remplace l'image dans `k8s/pod.yml`, puis pousse un
   commit `ci: deploy <sha> [skip ci]` sur la branche par défaut.

La compilation, les tests et le scan s'exécutent pour chaque push de branche,
pour les pull requests et lors d'un lancement manuel (`workflow_dispatch`).
La publication et la mise à jour du manifeste s'exécutent uniquement sur la
branche par défaut, hors pull request. Le workflow ne lance pas `kubectl` :
l'application du manifeste au cluster reste à la charge du déploiement/GitOps.

### Configuration du dépôt GitHub

- Placer ce projet à la racine du dépôt GitHub pour que le workflow soit détecté.
- Le nom de l'image est calculé automatiquement :
  `ghcr.io/<propriétaire>/<dépôt>:<sha-court>`, en minuscules.
- L'authentification utilise le `GITHUB_TOKEN` fourni automatiquement par GitHub,
  avec `packages: write` pour publier et `contents: write` pour pousser le manifeste.
  Aucun secret GitLab ni jeton personnel n'est nécessaire dans la configuration standard.
- Les règles de protection de la branche par défaut doivent autoriser le push
  du bot pour que `update-manifest` réussisse.
- Le premier passage réussi sur la branche par défaut remplace la référence
  au registre GitLab encore présente dans `k8s/pod.yml` par celle de GHCR.
- Si le package GHCR est privé, configurer les identifiants de lecture du registre
  dans le cluster Kubernetes (`imagePullSecrets`), ou rendre le package public.

Documentation : [publier dans GHCR avec GitHub Actions](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions)
et [action Trivy](https://github.com/aquasecurity/trivy-action).
