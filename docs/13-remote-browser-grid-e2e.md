# 13. Remote browser-grid e2e (ORES clusters, AWS + Hetzner)

**Issue:** [Doc 12](12-in-cluster-e2e.md) runs the browser suites with a browser
on the same machine as the test runner. But the ORES platform already operates
managed browser-automation servers on its AWS and Hetzner clusters — a shared,
maintained pool that's the right place to run UI e2e from CI without every
runner shipping Chromium. This doc is how the zed suites drive those remote
browser deployments, what's actually reachable, and the one architectural
constraint that shapes the whole thing.

## What the platform exposes (and what it doesn't)

`ORESoftware/k8s-cluster` deploys, in namespace `default` on both clouds
(`remote/argocd/dd-next-runtime/`):

- **`dd-browser-test-server`** (`:8104`) — a Fastify service that drives real
  Chromium through **all three back-ends** (Playwright, Puppeteer, Selenium)
  behind one declarative `POST /run` scenario API. This is the primary target.
- **`dd-selenium-server`** (`:8105`) — a standalone Selenium Grid + a Java
  scenario API.
- `dd-web-scraper` (`:8097`) and `dd-browser-job-runner` (`:8106`) — adjacent
  scraping / ephemeral-container-spawning services.

**There is no raw WebDriver/CDP endpoint exposed.** The Selenium Grid `:4444`
and any CDP `:9222` stay pod-internal — so you cannot point a stock Playwright
`connect()` / Selenium `RemoteWebDriver(url)` / Puppeteer `connect()` client at
a URL here. Automation goes through the `POST /run` DSL instead:
`goto`/`click`/`fill`/`waitForSelector`/`extractText`/`extractAttribute`/
`screenshot`/`press`/`select`/`waitForUrl` (see the service README). A run
returns `{ ok, finalUrl, finalTitle, extracted, screenshots, pageErrors }`.

(The zed harness still keeps env-gated `connect()` support —
`PW_CONNECT_WS` / `PUPPETEER_BROWSER_WS` / `SELENIUM_REMOTE_URL` in
[`playwright.config.ts`](https://github.com/zed-pkg/zed-e2e) and the
Puppeteer/Selenium specs — for the day a raw grid *is* exposed. It is inert
against the ORES `/run` servers.)

## Reaching it: gateway auth, and the in-cluster `/run` rule

- **Public, via the gateway.** Both clouds route the service under
  `/browser-test/*`, gated by the operator `Auth` header (local env
  `ALL_DOGS`); the gateway injects the inner `x-server-auth`. Reachable:
  `https://98.90.186.114/browser-test/...` (AWS node IP) and
  `https://hello.95-217-171-250.sslip.io/browser-test/...` (Hetzner). **But the
  gateway only exposes the GET diagnostics** (`/healthz`, `/tools`, `/status`,
  `/metrics`) — a `POST /browser-test/run` is `404` (the prefix isn't stripped
  and the service registers `/run` only at the bare path).
- **`POST /run` is in-cluster only.** Drive it from inside the cluster, e.g. by
  exec'ing the pod (it has Node 22 and its own `SERVER_AUTH_SECRET`):

  ```bash
  kubectl --context dd-ec2-runtime -n default exec deploy/dd-browser-test-server -- \
    node /dev/stdin <<'JS'
  const body = JSON.stringify({ tool: "playwright", steps: [
    { action: "goto", url: "https://example.com", waitUntil: "load" },
    { action: "waitForSelector", selector: "h1" },
    { action: "extractText", selector: "h1", name: "headline" } ] });
  fetch("http://localhost:8104/run", { method: "POST",
    headers: { "content-type": "application/json", "x-server-auth": process.env.SERVER_AUTH_SECRET },
    body }).then(r => r.json()).then(j => console.log(j.ok, j.extracted));
  JS
  ```

  Cluster access: **AWS** is a single kubeadm node reachable via the
  `dd-ec2-runtime` kube context (creds from `~/.aws`), or SSM Run Command on
  `i-0cc2461a55d491af6` when off-VPN. **Hetzner** is `ssh root@<ip>` with
  `~/.ssh/id_hetzner`, on-node `KUBECONFIG=/etc/kubernetes/admin.conf`.

## The constraint that shapes everything: the browser must reach the target

`/run` navigates *from inside the cluster*. So the URL under test must be
reachable **from that cluster** — a public URL, the cluster's own gateway, or an
in-cluster Service (ClusterDNS). A zed stack running in a local `kind` cluster
([doc 12](12-in-cluster-e2e.md)) is **not** reachable from AWS/Hetzner.
Therefore, to exercise the **zed UI** through the remote grid, the zed stack
must run in (or be reachable from) the same cluster as the browser server. The
in-memory profile deploys there as a ClusterIP service; the grid then points at
`http://dd-zed-web-server.<ns>.svc.cluster.local:8081`.

## Runner + scenarios

[`cluster/remote-grid.sh`](https://github.com/zed-pkg/zed-e2e/blob/main/cluster/remote-grid.sh)
codifies the in-cluster driver: given a kube context and a target web URL, it
POSTs the zed UI scenarios in
[`cluster/remote/scenarios.json`](https://github.com/zed-pkg/zed-e2e/blob/main/cluster/remote)
through the pod for **each** of playwright / puppeteer / selenium and asserts
the extracted text + `finalTitle`. The scenarios mirror the local `web-ui`
checks: the home recency list renders, HTMX search returns a match, a package
page shows the `zed add …` snippet, and the security headers are present.

## Status: partially verified

- **AWS — verified.** The `dd-browser-test-server` is healthy (`/healthz`,
  `/tools` → Playwright 1.56, Puppeteer 24.43, Selenium 4.44), and an in-cluster
  `POST /run` executed a real navigation-and-extract scenario through **all
  three** back-ends (example.com → `<h1>` "Example Domain": Playwright ~108ms,
  Puppeteer ~167ms, Selenium ~640ms). Gateway auth (`ALL_DOGS`) works for the
  diagnostics.
- **Hetzner — grid currently down.** The gateway is reachable and auth-gated,
  but `/browser-test/*` returns `502` (the `dd-browser-test-server` upstream is
  unhealthy), and node SSH `:22` was refused from the test host, so in-cluster
  `/run` could not be exercised. The mechanism is identical to AWS (same synced
  manifests) once the deployment is healthy.
- **zed UI through the grid — pending a cluster-reachable zed.** The runner and
  scenarios are committed; driving them against the zed UI requires the zed
  stack deployed into (or reachable from) the target cluster per the constraint
  above.
