# leartech-sp-control-umbrella

One reviewed release that deploys **SP's own control plane** into the
`ship-proven` namespace, authed to the **sp-infra** world (its issuer, its Hydra,
its credentials).

| in this umbrella (ship-proven ns) | NOT here |
|---|---|
| subscription-api (plans, per-cluster subscriptions, budget ledger, promos, top-ups) — and, later, cluster-creation / fleet provisioning | the product (MCP host, controller, plan/artifact/…) — that is `leartech-ship-proven-umbrella`; real infra — that is `leartech-infra-umbrella` |

## The boundary

There are three kinds of thing Ship Proven runs:

| kind | umbrella | reaches customer clusters? |
|---|---|---|
| real infra (auth/Hydra, Postgres, Mongo, redis) | `leartech-infra-umbrella` | **yes** |
| the product (MCP host, ba-service, controller, the APIs, sp-web, console) | `leartech-ship-proven-umbrella` | **yes** |
| **SP's control plane** (subscriptions, cluster creation) | **`leartech-sp-control-umbrella`** | **no** |

**The rule:** if a customer would run it — because their Controller + Agents do
the work it supports — it is product and goes over via the ship-proven umbrella.
If **only SP HQ** runs it to operate the fleet or the business, it belongs here
and stays. This should be a *small* list.

`leartech-subscription-api` is the first member: a subscription IS a cluster, and
the service that mints subscriptions, owns the budget ledger and pushes budgets
down to each cluster's gateway is something SP runs for itself. Customers do not
run their own subscription service.

## Why the ship-proven namespace and the sp-infra world

Same namespace, same world, same Hydra, same ClusterSecretStore as
`leartech-ship-proven-umbrella` — deliberately, so there is **zero new infra and
zero new Hydra**:

- the DB rides the existing `auth-postgresql` (a new `subscription_api` database +
  role, the console-api pattern);
- the inbound verifier points at the existing sp-infra issuer;
- the s2s client is minted by the existing sp-infra Hydra;
- Maestro (which delivers `user.onboarded`) is in ship-proven, so the consumer
  registration is a plain in-namespace Service.

The ONLY difference from the ship-proven umbrella is the promotion path: this
chart is released ONLY to SP's own cluster (the current GitOps repo,
`helmfiles/sp-control`) and **never promoted to a customer cluster**. The
ship-proven and infra umbrellas are moving to the managed Argo GitOps repo and
are promoted out; this one stays on the current process precisely because it must
not ride that path.

## Release pattern

The shared `tasks/helm/platform-release.yaml@main` task — the same
next-version + OCI publish + namespace-aware GitOps bump the other umbrellas use —
with `PLATFORM_PROMOTE_NAMESPACE=ship-proven` and
`PLATFORM_PROMOTE_HELMFILE=helmfiles/sp-control/helmfile.yaml`.

## Status

Skeleton. The chart (subscription-api dep + the sp-infra values wiring) lands
through the estate gates. Deploy prerequisites tracked in the onboarding
workspace's `SP-CONTROL-UMBRELLA-DESIGN.md`:

1. companion subscription-api chart PR — add the goose migrations Job (the repo
   has `migrations/0001_initial.sql` but the chart has no job to apply it);
2. GitOps on `sp-cluster-gitops` — `helmfiles/sp-control/` release; the
   `subscription_api` DB row + derived DSN; the `leartech-subscription-api-db`
   ExternalSecret in `ship-proven-resources`; bump the infra umbrella's
   auth-service pin to 0.1.199 (the DCR scopes + s2s client);
3. wire the MCP — `subscriptionApiUrl` on `leartech-mcp-servers` in the
   ship-proven umbrella (released as 0.0.195, which carries the key);
4. boot → e2e.
