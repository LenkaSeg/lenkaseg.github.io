---
layout: post
title: Onboarding a Forgejo-hosted project to Fedora Konflux
comments: true
---

# Onboarding a Forgejo-hosted project to Fedora Konflux

So you want to build container images on the Fedora Konflux instance? This
guide walks through the full process we followed to onboard a Codeberg-hosted
repository. It covers everything from getting access to the GitLab config repo
all the way to seeing your first images land on Quay — including the pitfalls
we ran into along the way.

## Useful links

Before we start, bookmark these:

- **Konflux UI**: https://konflux-ci.fedoraproject.org
- **Cluster Console**: https://console-openshift-console.apps.kflux-fedora-01.84db.p1.openshiftapps.com
- **Cluster API** (for `oc login`): https://api.kflux-fedora-01.84db.p1.openshiftapps.com:6443

## Step 1: Get access to the tenants-config GitLab repo

All Konflux tenant configuration is managed through GitOps in the
[tenants-config](https://gitlab.com/fedora/infrastructure/konflux/tenants-config)
repo on GitLab. The Konflux UI is read-only for configuration — everything
happens via merge requests.

To get started:

1. **Sign in via SAML SSO** at https://gitlab.com/groups/fedora/-/saml/sso to
   establish your group membership.
2. **Check your role** — Guest is enough to approve MRs, but you need at least
   Developer to merge them. Ask an existing maintainer to upgrade your role if
   needed.
3. **Get added to CODEOWNERS** for your tenant's directory so you can
   self-approve changes to your own files
   ([example MR](https://gitlab.com/fedora/infrastructure/konflux/tenants-config/-/merge_requests/179)).

A common gotcha here: being listed in `CODEOWNERS` does **not** grant you
GitLab membership. You need both — the CODEOWNERS entry for approval rights,
and actual project/group membership with the right role for merge permissions.

## Step 2: Create your tenant namespace

Follow the instructions in the
[tenants-config repo](https://gitlab.com/fedora/infrastructure/konflux/tenants-config/-/tree/main/clusters/kflux-fedora-01):

1. Run the **`create-tenant-resources`** playbook to generate tenant resources
   (namespace, RBAC, quota).

This produces three files:

- **`ns.yaml`** — your namespace with the `konflux-ci.dev/type: tenant` label
- **`rbac.yaml`** — a RoleBinding granting `konflux-admin-user-actions` to your
  FAS user
- **`kustomization.yaml`** — ties together the quota, RBAC, namespace, and your
  applications directory. Quota can be increased later.

## Step 3: Define your Applications and Components

This is where you describe *what* Konflux should build. We used a
Kustomize-based Configuration-as-Code approach with a layered structure:

- **Shared base** — a generic Component template containing the git URL,
  provider annotations (`git-provider: forgejo`,
  `git-provider-url: https://codeberg.org`), pipeline config, and the
  `build.appstudio.openshift.io/request: configure-pac` annotation.
- **Per-application base** — an Application CR plus per-variant component
  overrides (e.g. rawhide vs. stable).
- **Per-environment overlays** (staging/production) — patches for application
  names, component names, context paths, and Dockerfile paths.

Run the **`update-tenant-apps`** playbook to generate an ArgoCD application
manifest per tenant directory and update the ArgoCD kustomization.

Open a merge request with all of this
([example MR](https://gitlab.com/fedora/infrastructure/konflux/tenants-config/-/merge_requests/176)).

## Step 4: Add ImageRepository resources

Each Component needs a
corresponding `ImageRepository` CR. Without it, the image controller never
provisions a Quay registry repo, `spec.containerImage` never gets set on your
Component, and the build service stays permanently blocked — no webhook, no
PaC PR, no builds.

An ImageRepository is a simple resource. No kustomize layering needed — just
flat YAML files:

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ImageRepository
metadata:
  name: forge-rawhide-production
  namespace: fedora-infra-tenant
  annotations:
    image-controller.appstudio.redhat.com/update-component-image: "true"
  labels:
    appstudio.redhat.com/application: forge-production
    appstudio.redhat.com/component: forge-rawhide-production
spec:
  image:
    name: fedora-infra-tenant/forge-rawhide-production
    visibility: public
```

The key annotation `update-component-image: "true"` tells the image controller
to write the provisioned Quay URL back into `spec.containerImage` on the
matching Component. That's what unblocks everything.

Open a separate MR for these
([example MR](https://gitlab.com/fedora/infrastructure/konflux/tenants-config/-/merge_requests/178)).

## Step 5: Create the SCM secret

**Before** merging your application/component MRs, create a source control
secret in the cluster so Konflux can authenticate with your Forgejo/Codeberg
instance:

```bash
oc create secret generic pipelines-as-code-codeberg \
  -n {namespace} \
  --type=kubernetes.io/basic-auth \
  --from-literal=password={FORGEJO_TOKEN}

oc label secret pipelines-as-code-codeberg -n {namespace} \
  appstudio.redhat.com/credentials=scm \
  appstudio.redhat.com/scm.host=codeberg.org
```

When creating the Forgejo/Codeberg access token, the
[Konflux docs](https://konflux-ci.dev/docs/building/creating-forgejo/)
specify these scopes:

| Scope | Permission |
|-------|------------|
| issue | Read and Write |
| organization | Read |
| repository | Read and Write |
| user | Read |

**Important**: do not restrict the token to a specific repository — scopes
like `write:user` are incompatible with repo-scoped tokens on Forgejo. If the
documented scopes don't work, try a full-permissions token. We hit a case
where the build-service's `forgejo-sdk v2` couldn't correctly validate token
scopes against a newer Forgejo version, and a broader token was the only
workaround.

## Step 6: Merge and verify

Once the secret is in place, merge your MRs. ArgoCD will sync the resources to
the cluster. Give it a few minutes, then verify:

```bash
# Check that containerImage got populated on each Component
oc get components -n {namespace} \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containerImage'

# Check that ImageRepositories are ready
oc get imagerepositories -n {namespace} \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state'

# Check PaC configuration status
oc get components -n {namespace} \
  -o custom-columns='NAME:.metadata.name,STATUS:.metadata.annotations.build\.appstudio\.openshift\.io/status'
```

If `spec.containerImage` is populated and the status shows
`"state":"enabled"`, PaC is configured and you're in business.

## Step 7: Handle the PaC pull requests

Konflux will open pull requests on your source repo with auto-generated Tekton
pipeline files in `.tekton/`. You have two options:

1. **Merge them as-is** to use Konflux's default pipeline configuration.
2. **Close them and use handcrafted pipelines** — if you already have `.tekton/`
   files, update them to match the new Konflux setup:
   - Application/component labels matching your Konflux component names
   - `output-image` pointing to
     `quay.io/redhat-user-workloads/{namespace}/{component}`
   - Latest task bundle versions (copy the `@sha256:...` references from the
     Konflux-generated files)
   - `serviceAccountName: build-pipeline-{component}`

We went with option 2 — we had existing pipelines with custom version tagging
logic that we wanted to keep, so we compared the auto-generated files with ours,
adopted the updated task bundles and correct labels, and kept our customizations.

## Step 8: Re-triggering if something goes wrong

The `configure-pac` annotation gets consumed by the build service on the first
attempt. If it fails (e.g. due to a token scope error), re-add it to retry:

```bash
# Single component
oc annotate component {component} -n {namespace} \
  build.appstudio.openshift.io/request=configure-pac --overwrite

# All components at once
for comp in $(oc get components -n {namespace} -o name); do
  oc annotate $comp -n {namespace} \
    build.appstudio.openshift.io/request=configure-pac --overwrite
done
```

## Gotchas we hit

Here's a summary of the non-obvious issues we ran into — save yourself some
debugging time:

- **ImageRepository is mandatory** when the image controller is deployed.
  Without it, Components stay in "waiting for spec.containerImage" limbo
  forever. The docs don't make it obvious that this is a prerequisite for PaC
  setup.

- **Forgejo token scope validation can fail** due to an SDK version mismatch
  between the build-service's `forgejo-sdk v2` and newer Codeberg/Forgejo
  versions. The error message says "Forgejo access token does not have enough
  scope" even when the token has all the right scopes. A full-permissions token
  is the workaround.

- **GitLab CODEOWNERS does not equal membership.** Being listed in CODEOWNERS
  lets you approve MRs, but you still need direct project/group membership with
  at least the Developer role to actually merge. Guest role is not sufficient
  for merging on protected branches.

- **Rate limiting when triggering PaC on many components.** If you annotate all
  your components at once, some may fail because Codeberg's API rate limit gets
  hit. Retry the failed ones individually with a short pause between each.

---

*Written by lenkaseg and Claude (Anthropic), May 2026, based on the onboarding experience
of the `fedora-infra-tenant` to the Fedora Konflux instance.*
