---
name: update-container-images
description: >
  Update pinned container image tags in Dockerfiles (Dockerfile, Dockerfile.test, Dockerfile.<anything>, etc.)
  and compose files (compose.yml, compose.yaml, docker-compose.yml, docker-compose.yaml, at any depth)
  to the latest version compatible with each image's current versioning scheme,
  using the source and registry URLs documented near each service or FROM line where available,
  and otherwise deriving the registry directly from the image reference itself.
  Use when the user asks to bump, update, upgrade, refresh, or check Docker/container image versions
  in a Dockerfile or compose file, or asks "are our images up-to-date" for this repo.
---

# Update Dockerfile and Docker Compose image tags

Services in a compose file are sometimes documented with a comment block right above them, e.g.:

```yaml
# Loki – lightweight log storage
#
#   https://github.com/grafana/loki
#   https://hub.docker.com/r/grafana/loki
#
loki:
  image: grafana/loki:3.7.6
```

When present, that comment block is a helpful source of truth for "what is this image and where do I check its versions";
use it instead of guessing at a registry from the image string alone,
since the same org can publish under different names in different registries.
But don't assume every repo follows this convention: plenty of compose files and Dockerfiles have no comments at all.
In that case, derive the registry directly from the image reference itself (see step 2).
The comment is a nice-to-have cross-check and a source of changelog links, not a prerequisite for finding the latest tag.

Dockerfiles pin base images the same way, via `FROM` lines,
sometimes with a registry URL comment right above, sometimes with no per-line comment at all.
The same update, same care about versioning scheme, and same report apply to both file types.
Treat every `FROM <image>:<tag>` line as equivalent to a compose service's `image:` line, with the differences noted below.

## 1. Find the target(s)

  - If invoked with an argument that looks like a path or glob, scope to that file (or files) only.
  - Otherwise, recursively find every Dockerfile and every compose file anywhere under the current directory
    (any depth, not just the top level):

    ```
    find . -type f \( \
      -name 'compose.yml' -o -name 'compose.yaml' -o \
      -name 'docker-compose.yml' -o -name 'docker-compose.yaml' \
    \)
    find . -type f \( -name 'Dockerfile' -o -name 'Dockerfile.*' -o -name '*.Dockerfile' \)
    ```

    This covers plain `Dockerfile`s as well as suffixed/prefixed variants like `Dockerfile.test`, etc.
    Skip directories like `.git`, `node_modules`, and `vendor`.
    Confirm the list with the user before touching more than one file.
  - If an argument contains `--dry-run`, do the full analysis and report but skip step 4 (no file edits).

## 2. Parse each service / FROM line

For every `FROM` instruction in a Dockerfile, and every top-level entry under `services:` in a compose file:

  - In compose files, skip entries with no `image:` key (e.g. a service defined only with `build:`) —
    there's nothing to bump.
  - In Dockerfiles, skip a `FROM` line whose image name matches an earlier stage's `AS <name>`
    alias in the same file (e.g. a final `FROM builder AS final` or `COPY --from=builder`) —
    that references a build stage, not a registry image.
    Multi-stage builds otherwise get every real `FROM` line checked independently, since stages
    commonly pin unrelated images (e.g. `golang` for the builder stage, `alpine` for the final stage).
  - Determine the registry to query, in this order of preference:
    1. If there's a comment block immediately above the entry (up to the previous blank/non-comment line),
       pull out the **source repo** URL (`github.com/<owner>/<repo>`, for release-notes context) and the
       **registry** URL — one of `hub.docker.com/r/<ns>/<repo>`, `quay.io/repository/<ns>/<repo>`, or a
       GitHub Packages link (`github.com/<owner>/<repo>/pkgs/container/<repo>`, i.e. `ghcr.io/<owner>/<repo>`).
       Some entries only document the source repo — that's fine, fall back to it for step 3.
    2. Otherwise, derive the registry directly from the image reference string,
       which is almost always enough on its own:
       - No registry host in the reference (e.g. `grafana/loki`, `golang`, `alpine`) → Docker Hub.
         With a namespace (`<ns>/<repo>`) → `hub.docker.com/r/<ns>/<repo>`.
         With no namespace (bare `<repo>`, e.g. `golang`, `alpine`, `node`, `python`) → it's a Docker Official Image:
         `hub.docker.com/_/<repo>`, no source repo to fall back on.
       - A registry host is present as the first path segment before the first `/`
         (i.e. it contains a `.` or `:`, or is `localhost`) — use that host directly, e.g.
         `ghcr.io/<owner>/<repo>`, `quay.io/<ns>/<repo>`, `registry.k8s.io/<ns>/<repo>`, or any other custom registry host.
         The owner/namespace and repo are already right there in the reference.
         That's enough to look up tags in step 3 — don't skip an image just because it's undocumented.
  - Parse the current tag from the `image:` line or `FROM` line and infer its versioning scheme:
    the prefix (`v` or none) and the number of numeric segments (most images use `v?MAJOR.MINOR.PATCH`,
    but check the actual tag in front of you rather than assuming).
    An image that has always been tagged `v1.2.3` should stay `vX.Y.Z`, not jump to a `1.2.3-alpine` or `nightly` variant.
    A base image tagged `1.26.5-alpine` should stay on the `-alpine` variant, not jump to a bare tag.

## 3. Find the latest matching tag

Prefer querying the registry directly over scraping the human-facing web page.
Registry pages are JS-rendered and easy to misread, while the underlying APIs return exact, machine-readable tag lists.
The `image:`/`FROM` reference already tells you the registry host and repo path, so derive the API call from that
(any comment-block URLs are your cross-check and your source for release notes, not a hard requirement to fetch as HTML).

Use `curl` via Bash. Registry-specific list-tags calls:

  - **Docker Hub** (`docker.io`, e.g. `grafana/loki`, `otel/opentelemetry-collector-contrib`):

    ```
    curl -s "https://hub.docker.com/v2/repositories/<ns>/<repo>/tags?page_size=100" | jq -r '.results[].name'
    ```

    Paginate via the `next` field in the response if you need more than 100.
    For a Docker Official Image (unnamespaced, e.g. `golang`, `alpine`), use `library` as `<ns>`.

  - **Quay.io** (e.g. `quay.io/prometheus/node-exporter`):

    ```
    curl -s "https://quay.io/api/v1/repository/<ns>/<repo>/tag/?limit=100&onlyActiveTags=true" | jq -r '.tags[].name'
    ```

  - **GHCR / GitHub Packages** (e.g. `ghcr.io/google/cadvisor`):
    GHCR's tag-list API needs auth even for public images, so instead use the OCI Distribution v2 API anonymously:

    ```
    TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:<owner>/<repo>:pull" | jq -r .token)
    curl -s -H "Authorization: Bearer $TOKEN" "https://ghcr.io/v2/<owner>/<repo>/tags/list" | jq -r '.tags[]'
    ```

  - **Any other registry host** (e.g. `registry.k8s.io/...`, `cr.fluentbit.io/fluent/fluent-bit`,
    a private/self-hosted registry): try the same anonymous OCI v2 flow against that host before falling back to GitHub tags:

    ```
    TOKEN=$(curl -s "https://<registry-host>/token?scope=repository:<ns>/<repo>:pull" | jq -r .token)
    curl -s -H "Authorization: Bearer $TOKEN" "https://<registry-host>/v2/<ns>/<repo>/tags/list" | jq -r '.tags[]'
    ```

    Not every host implements the same auth handshake — if the token endpoint 404s or the registry needs real credentials,
    fall back to the GitHub tags proxy below rather than guessing at auth.

  - **Fallback for any image** (registry API unreachable, unauthenticated, or undocumented):
    if you know (from a comment or from the image path itself, e.g. `ghcr.io/<owner>/<repo>`) the upstream GitHub repo,
    use its tags as a proxy, since these projects typically publish images matching their git tags:

    ```
    curl -s "https://api.github.com/repos/<owner>/<repo>/tags?per_page=100" | jq -r '.[].name'
    ```

    If `GITHUB_TOKEN` is set in the environment,
    pass it as `-H "Authorization: Bearer $GITHUB_TOKEN"` to avoid the unauthenticated rate limit.

Once you have a raw tag list:

  1. Filter to tags matching the current tag's shape from step 2 (same prefix, same number of numeric segments,
     no extra suffix like `-rc`, `-beta`, `-alpine`, no `latest`, `main`, `nightly`, date-stamped tags)
     Unless the current tag already uses that kind of suffix, in which case match it.
  2. Sort the survivors as semantic versions (numeric major.minor.patch comparison, not string sort) and take the highest.
  3. If nothing survives the filter, don't guess — leave that service's image untouched
     and note it in the report as needing manual review (the upstream project may have changed its tagging scheme).

## 4. Apply the change

Edit only the tag in the `image:` line or `FROM` line, in place, preserving indentation,
the rest of the file, and (in Dockerfiles) any trailing `AS <stage>` untouched.
Don't touch `build:`-only services, don't reorder services or stages, don't "clean up" unrelated lines.

If the new major version differs from the current major version, still apply it (that's genuinely the latest),
but flag it clearly in the report below — a major bump is more likely to need config changes
and is worth the user reading the release notes for.

## 5. Report

Finish with a table like:

| File | Service / stage | Old tag | New tag | Notes |
|----|----|----|----|----|
| observability/compose.yaml | loki | 3.7.6 | 3.8.0 | |
| observability/compose.yaml | mimir | 3.1.4 | 3.1.4 | already latest |
| observability/compose.yaml | grafana | 13.1.3 | 14.0.0 | **major bump** — check release notes |
| services/ingest/compose.yml | fluent-bit | 5.1.0 | 5.1.0 | couldn't confirm — registry API unreachable, left unchanged |
| user-service/Dockerfile | builder (golang) | 1.26.2 | 1.26.5 | |
| user-service/Dockerfile | final (alpine) | 3.23.4 | 3.24.1 | |
| user-service/Dockerfile.test | final (golang) | 1.26.2 | 1.26.5 | |

Then run `git diff` in the affected repo so the changes are visible, and stop.
Don't commit, push, or open a PR unless the user explicitly asks for that next.
