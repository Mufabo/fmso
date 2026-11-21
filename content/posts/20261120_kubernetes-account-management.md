---
title: "Account Management in Kubernetes"
date: 2026-11-20
draft: true
description: "How Kubernetes handles user and service accounts: what it manages and what it doesn't, client certificates, OIDC with structured authentication configuration, managed cloud identities, CI without stored secrets, groups, anonymous access, and how to onboard, review and offboard accounts."
summary: "Kubernetes has no User object and no 'add user' command. Here is how accounts actually work, which authentication method to pick, and how to run the whole lifecycle from onboarding to offboarding."
tags: ["kubernetes", "security", "authentication", "oidc", "rbac", "service accounts", "identity"]
categories: ["Programming"]
---

The first time you try to give a colleague access to a Kubernetes cluster, you look for something like `kubectl create user`. It doesn't exist. Kubernetes has no User object, no password database and no "invite" button. Yet every request to the API server is made *by someone*, and every audit log entry has a username in it.

This post explains how that fits together: what an account is in Kubernetes, which parts Kubernetes manages and which it leaves to you, the ways a person can log in, how machines and CI pipelines get an identity, and how to run the full lifecycle: onboarding, access reviews and offboarding.

Two earlier posts cover neighbouring topics in depth, so this one only touches them: [Using Kubernetes Service Account Tokens]({{< relref "20261107_kubernetes-service-account-tokens.md" >}}) for workload identity, and [Authorization in Kubernetes]({{< relref "20261116_authorization-in-kubernetes.md" >}}) for what an identity is allowed to do once it is known.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## Two kinds of accounts

Every Kubernetes cluster has two categories of users: **service accounts**, which Kubernetes manages, and **normal users**, which it doesn't[^1].

| | Normal users | Service accounts |
| --- | --- | --- |
| Meant for | Humans (and some external systems) | Processes: Pods, controllers, CI |
| Stored in Kubernetes? | **No.** There is no User object. | Yes, a namespaced `ServiceAccount` object. |
| Created by | Whatever issues the credential: a CA, an identity provider, a cloud IAM | `kubectl create serviceaccount` |
| Credential | Client certificate, OIDC token, cloud token, ... | Service account token (a signed JWT) |
| Username | Whatever the credential says, e.g. `jane` or `oidc:jane@example.com` | `system:serviceaccount:<namespace>:<name>` |

The important sentence in the documentation is that normal users **cannot be added through an API call**[^1]. Kubernetes trusts an outside system to vouch for them. If a request carries a valid certificate signed by the cluster CA, or a valid token from a trusted identity provider, the API server accepts the username written inside it, whether or not anyone has ever heard of that user before.

So "account management" for humans in Kubernetes really means two things:

1. **Managing the identity source**: the CA, identity provider or cloud IAM that issues credentials.
2. **Managing RBAC**: the Roles and bindings that decide what each name or group may do.

Kubernetes connects the two, but only stores the second.

## What an identity looks like

Whatever method authenticated a request, the result is always the same small set of attributes[^1]:

| Attribute | Example | Notes |
| --- | --- | --- |
| username | `jane@example.com` | A string. Kubernetes doesn't check that it means anything. |
| uid | `8f2b...` | Optional. More stable than a username, which can be reused. |
| groups | `["shop-team", "system:authenticated"]` | The main thing you bind RBAC to. |
| extra | `{"scopes": ["admin"]}` | Optional key/values for authorizers and webhooks. |

A few groups are added or reserved by Kubernetes itself:

| Group | Who is in it |
| --- | --- |
| `system:authenticated` | Every successfully authenticated identity. |
| `system:unauthenticated` | Anonymous requests. |
| `system:serviceaccounts` | All service accounts. |
| `system:serviceaccounts:<namespace>` | All service accounts in one namespace. |
| `system:masters` | Bypasses authorization completely. Break-glass only. |

Anything granted to `system:authenticated` is granted to every human, every service account and every CI job in the cluster. Treat bindings to it as a public announcement.

You can always ask the API server who it thinks you are:

```bash
kubectl auth whoami
# ATTRIBUTE   VALUE
# Username    oidc:jane@example.com
# Groups      [oidc:shop-team system:authenticated]
```

## How authentication works

The API server runs a list of **authenticators**. Each one looks at the request: a client certificate, an `Authorization: Bearer` header, proxy headers. The first one that recognises the credential wins. If a credential is present but invalid, the request fails with `401 Unauthorized`. If there is no credential at all, the request becomes **anonymous**, with the username `system:anonymous` and the group `system:unauthenticated`[^1].

The authenticators you can enable:

| Method | Typical use | Revocation |
| --- | --- | --- |
| X.509 client certificates | Admins, bootstrap, small clusters | None until expiry |
| JWT / OIDC (structured authentication configuration) | Humans via SSO, CI via federation | Disable in the IdP; tokens die at `exp` |
| Webhook token authentication | Custom or enterprise identity systems | Immediate (after cache TTL) |
| Authenticating proxy | Pinniped, Teleport, OpenShift-style gateways | At the proxy |
| Service account tokens | Workloads | Bound tokens die with their Pod |
| Static token file | Nothing in production | Edit the file and restart the API server |

The rest of this post goes through them, starting with the one every cluster has.

## Client certificates

Every cluster has a CA, and any client certificate signed by it is accepted. The certificate's **Common Name** becomes the username and each **Organization** becomes a group. A certificate for `CN=Ada Lovelace,O=Users,O=Staff,O=Programmers` authenticates as the user `Ada Lovelace` in the groups `Users`, `Staff` and `Programmers`[^1].

### Issuing a certificate with the CSR API

You don't need direct access to the CA key. Kubernetes has a **CertificateSigningRequest** API and a built-in signer, `kubernetes.io/kube-apiserver-client`, for exactly this[^2]:

```bash
# 1. The user creates a key and a signing request
openssl genrsa -out jane.key 3072
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=shop-team"

# 2. Submit it to the cluster
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(base64 -w0 < jane.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400   # one day
  usages:
    - client auth
EOF

# 3. An admin approves it
kubectl certificate approve jane

# 4. Fetch the signed certificate and build a kubeconfig entry
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
kubectl config set-credentials jane \
  --client-key=jane.key --client-certificate=jane.crt --embed-certs=true
```

The private key never leaves Jane's machine; only the CSR is sent. `expirationSeconds` (minimum 600, i.e. ten minutes) sets the lifetime, capped by the controller manager's `--cluster-signing-duration`[^2]. On kubeadm clusters, `kubeadm kubeconfig user --client-name=jane` does the same thing in one step, but it uses the CA key directly on a control plane node.

### Why certificates are a poor fit for people

Certificates are simple and need no extra infrastructure, which is why they're the default for the first admin. For a team, they have serious problems.

**They can't be revoked.** The documentation is blunt: Kubernetes does not support certificate revocation, and any issued certificate remains valid until it expires[^1]. If Jane leaves with a one-year certificate, she has a one-year credential. You can delete RBAC bindings to `jane`, but not to her groups if colleagues share them, and the only complete fix is rotating the cluster CA.

**Groups are baked in.** Moving Jane to another team means issuing a new certificate. Her old one still carries `O=shop-team` until it expires.

**Approving CSRs is an admin power.** The `kube-apiserver-client` signer doesn't restrict the subject. Anyone who can create and approve CSRs for it can mint a certificate for any name, including the names of system components or `O=system:masters`[^6]. Treat `approve` on `certificatesigningrequests/approval` like cluster-admin.

If you do use certificates for people, keep them short-lived (hours or days, not years), never put users in `system:masters`, and plan to move to OIDC.

## OIDC and structured authentication configuration

For humans, the right answer in almost every self-managed cluster is to let an **identity provider** handle accounts (Keycloak, Dex, Entra ID, Okta, Google, Authentik) and have the API server accept the JWTs it issues. Users, passwords, MFA, groups and offboarding then live in one place, the same place as for every other company system.

For years this was configured with `--oidc-*` flags on the API server, which allowed exactly one provider and very little control over how claims became usernames. The **AuthenticationConfiguration** file replaces them. It went alpha in v1.29 and is **stable since v1.34**[^1][^3]. You pass it with `--authentication-config`; combining it with any `--oidc-*` flag is a misconfiguration and the API server refuses to start[^1].

### A configuration for an SSO provider

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
jwt:
  - issuer:
      url: https://sso.example.com/realms/platform
      audiences:
        - kubernetes
    # Reject tokens that don't meet our rules, before mapping anything.
    claimValidationRules:
      - expression: "claims.email_verified == true"
        message: "email address must be verified"
    # Turn claims into a Kubernetes identity.
    claimMappings:
      username:
        expression: '"oidc:" + claims.email'
      groups:
        expression: 'claims.groups.map(g, "oidc:" + g)'
      uid:
        claim: sub
    # Final check on the resulting identity.
    userValidationRules:
      - expression: "!user.username.startsWith('system:')"
        message: "username must not use the reserved system: prefix"
```

What each part does[^1]:

- **`issuer`**: where tokens come from and which audience (`aud`) they must have. The API server fetches the provider's discovery document and signing keys from here. If the provider isn't reachable from the control plane, `discoveryURL` and `certificateAuthority` let you point elsewhere.
- **`claimValidationRules`**: CEL expressions over the raw token claims. All must be true, or the token is rejected.
- **`claimMappings`**: how to build `username`, `groups`, `uid` and `extra`, either from a single claim with a prefix or from a CEL expression.
- **`userValidationRules`**: CEL expressions over the final identity.

**Use a prefix.** Without one, a user whose IdP name happens to be `system:kube-controller-manager`, or a group called `system:masters`, would be treated as that identity. Prefixes such as `oidc:` keep IdP names in their own namespace, and the `userValidationRules` above block the reserved prefix as a second line of defence.

Three more things the file gives you that the flags never did:

- **Multiple issuers.** Up to 64 JWT authenticators, each with a unique issuer URL[^1]. One for employee SSO, one for CI, one for a partner's IdP.
- **Automatic reload.** The API server watches the file and reloads authenticators when it changes, no restart needed. Write it atomically (write a temp file, then rename)[^1].
- **Anonymous access settings**, covered further down, in the same file.

### Logging in from kubectl

`kubectl` doesn't speak OIDC itself. It runs a **credential plugin**, an external command that returns a token, through the `exec` section of the kubeconfig. This mechanism is stable since v1.22[^1]. The most common plugin is [kubelogin](https://github.com/int128/kubelogin)[^7], installed as `kubectl oidc-login`:

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: prod
    cluster:
      server: https://k8s.example.com:6443
      certificate-authority-data: LS0tLS1CRUdJTi...
users:
  - name: sso
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=https://sso.example.com/realms/platform
          - --oidc-client-id=kubernetes
        interactiveMode: IfAvailable
contexts:
  - name: prod
    context:
      cluster: prod
      user: sso
current-context: prod
```

The first `kubectl` command opens a browser for SSO login. After that, the plugin caches the token and refreshes it as needed. The nice side effect: this kubeconfig contains **no secret at all**, so you can put it in a wiki or hand it to every new team member.

### Token lifetime is your revocation window

The API server validates OIDC tokens offline, by checking the signature. It never asks the IdP whether the user still exists. Disabling Jane in the IdP stops her getting *new* tokens, but a token she already holds stays valid until its `exp`. Keep ID token lifetimes short, between five and fifteen minutes, so offboarding takes effect quickly. The refresh token, held by kubelogin, is what keeps a session alive, and the IdP *does* check that on every refresh.

## CI pipelines without stored secrets

A CI job that deploys to a cluster is a classic source of long-lived credentials: a kubeconfig with a certificate or service account token, pasted into the CI system's secret store years ago.

Because the AuthenticationConfiguration accepts several issuers, you can trust your CI provider's OIDC tokens directly. GitHub Actions, GitLab and most other CI systems issue a short-lived JWT to each job that describes the repository, branch and workflow[^8]:

```yaml
jwt:
  # ... the SSO issuer from above ...
  - issuer:
      url: https://token.actions.githubusercontent.com
      audiences:
        - k8s-prod
    claimValidationRules:
      - expression: 'claims.repository == "acme/shop" && claims.ref == "refs/heads/main"'
        message: "only the main branch of acme/shop may deploy"
    claimMappings:
      username:
        expression: '"github:" + claims.repository + ":" + claims.ref'
      groups:
        expression: '["github:deployers"]'
```

The job requests a token for audience `k8s-prod` and uses it as a bearer token:

```yaml
# .github/workflows/deploy.yml (excerpt)
permissions:
  id-token: write
  contents: read
steps:
  - name: Get a cluster token
    run: |
      TOKEN=$(curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
        "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=k8s-prod" | jq -r .value)
      echo "::add-mask::$TOKEN"
      kubectl config set-credentials ci --token="$TOKEN"
```

Then bind `github:deployers` to a deploy Role in the target namespace. Nothing is stored in the CI system, a token stolen from a log is worthless within minutes, and the claim rule means a fork or feature branch can't deploy. The API server must be able to reach the issuer's discovery endpoint for this to work.

## Managed clusters: cloud IAM as the identity source

On managed Kubernetes, the cloud provider usually runs the authenticator for you and maps its own IAM identities into Kubernetes:

- **EKS** maps IAM users and roles through **access entries**. The older `aws-auth` ConfigMap is deprecated, and AWS recommends migrating existing clusters to the access entry API[^5]. Access entries are managed through AWS APIs (and so Terraform or CloudFormation), and a broken ConfigMap can no longer lock everyone out.
- **GKE** authenticates Google accounts and service accounts. Both Google Cloud IAM and Kubernetes RBAC are checked, and Google Groups can be used as RBAC subjects.
- **AKS** integrates with **Microsoft Entra ID**, using kubelogin as the credential plugin, and can use either Azure RBAC or Kubernetes RBAC for authorization. AKS also lets you add external identity providers through the same upstream structured authentication configuration[^9].

The principle doesn't change: the cloud IAM is the identity source, and you still decide what those identities can do inside the cluster. Keep the mapping in code alongside the rest of your infrastructure.

## Other authenticators

**Webhook token authentication.** The API server sends each bearer token to an external service as a `TokenReview` and accepts the answer[^1]. Revocation is immediate apart from a short cache (two minutes by default). Use it when you have an identity system that doesn't issue JWTs.

**Authenticating proxy.** A proxy in front of the API server authenticates the user however it likes and forwards the request with headers such as `X-Remote-User` and `X-Remote-Group`. The API server only trusts those headers when the proxy presents a client certificate from a dedicated CA (`--requestheader-client-ca-file`)[^1]. Tools like Teleport and Pinniped use this or similar approaches to give one login across many clusters.

**Static token file.** A CSV of tokens and users passed with `--token-auth-file`. Tokens last indefinitely and the list can't change without restarting the API server[^1]. There is no good reason to use it outside a lab.

## Service accounts: identities for workloads

Service accounts are the one kind of account Kubernetes manages itself. They are cheap, namespaced, and meant to be created **one per application**. The details of their tokens are in the [service account tokens post]({{< relref "20261107_kubernetes-service-account-tokens.md" >}}). From an account management point of view, the rules are:

- **Never run workloads as `default`.** Every Pod that doesn't set `serviceAccountName` shares it, and so shares any permission someone ever bound to it.
- **Turn off token automounting** for workloads that don't call the Kubernetes API.
- **Don't use service accounts for people.** Shared "admin" service account tokens hide who did what in the audit log and can't be tied to a person's offboarding.
- **Prefer federation over stored tokens** for anything outside the cluster: the CI pattern above, or `kubectl create token` for short-lived needs.

## Anonymous access

By default, requests with no credentials are allowed through authentication as `system:anonymous`, and RBAC decides what they may do. On most clusters that is very little: health checks, `/version` and discovery. But a single mistaken binding to `system:unauthenticated` exposes that resource to the internet.

Since v1.34 you can restrict anonymous authentication to specific paths in the AuthenticationConfiguration[^1][^4]:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
anonymous:
  enabled: true
  conditions:
    - path: /livez
    - path: /readyz
    - path: /healthz
```

Every other endpoint now rejects anonymous requests, even if an RBAC binding would allow them. If you set `anonymous` in the file, you can't also use the `--anonymous-auth` flag[^1].

## Groups: the link between identity and RBAC

Whatever the authenticator, bind RBAC to **groups**, not to individual users. Let the IdP decide who is in `shop-team`; let Kubernetes decide what `shop-team` may do:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shop-team-edit
  namespace: shop
subjects:
  - kind: Group
    name: oidc:shop-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

With this in place, onboarding Jane to the shop team is a group membership change in the IdP. Nobody edits YAML, and nothing needs to be cleaned up in the cluster when she leaves.

A typical group design:

| IdP group | Bound to | Scope |
| --- | --- | --- |
| `oidc:platform-admins` | `admin` or a custom cluster role | Cluster-wide, via ClusterRoleBinding |
| `oidc:<team>` | `edit` | RoleBinding in the team's namespaces |
| `oidc:developers` | `view` | RoleBindings in shared namespaces |
| `oidc:auditors` | `view` plus read on RBAC objects | Cluster-wide, read-only |
| `github:deployers` | Custom deploy Role | Only the namespaces the pipeline deploys to |

The [authorization post]({{< relref "20261116_authorization-in-kubernetes.md" >}}) covers how these roles work and which permissions are more dangerous than they look.

## The account lifecycle

### Onboarding

With OIDC and group-based RBAC, onboarding is:

1. Add the person to the right IdP groups.
2. Send them the shared kubeconfig (it contains no secret).
3. They run `kubectl auth whoami` to check their username and groups.

With certificates, it means a CSR, an approval and a kubeconfig with a private key in it, for every person, every time it expires.

### Access reviews

Because Kubernetes doesn't store users, there is no list of "all accounts". You review access from the RBAC side:

```bash
# Who is bound to cluster-admin?
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | select(.roleRef.name=="cluster-admin")
  | .metadata.name + ": " + ([.subjects[]? | .kind + "/" + .name] | join(", "))'

# What can a given group do in a namespace?
kubectl auth can-i --list -n shop --as=nobody --as-group=oidc:shop-team

# Who can delete secrets in a namespace? (krew plugin)
kubectl who-can delete secrets -n shop
```

Community plugins like `kubectl who-can`, `rakkess` and `rbac-lookup` make these reviews faster. Then compare the subjects you find with the IdP: a binding to a user or group that no longer exists there is dead weight at best, and at worst ready to be claimed by a new account with the same name.

The **audit log** completes the picture: it records the username, groups and, for OIDC, the uid of every request, so "who changed this Deployment?" has an answer. This is another reason not to share accounts.

### Offboarding

How quickly an account stops working depends entirely on how it was created:

| Credential | How to cut access | How quickly it takes effect |
| --- | --- | --- |
| OIDC token | Disable the user in the IdP | When the current ID token expires (minutes, if you kept them short) |
| Cloud IAM | Remove the access entry or IAM binding | Almost immediately |
| Webhook token | Disable in the backing system | After the cache TTL |
| Client certificate | Remove user bindings; if the groups are shared, rotate the CA | Not until the certificate expires, unless you rotate the CA |
| Service account token | Delete the Pod, the bound object, or the service account | Immediately for bound tokens |
| Static token | Edit the file, restart the API server | After the restart |

The table is the strongest argument for OIDC or cloud IAM for humans: offboarding happens in the one system HR already removes people from.

### Break-glass access

You still need a way in when the IdP is down. Keep one emergency credential, typically a client certificate in a group bound to `cluster-admin` through RBAC, not `system:masters` (which can't be restricted at all). Store it offline or in a vault with access alerts, give it a limited lifetime, and treat every use as an incident that ends with issuing a new one.

## Kubeconfig hygiene

The kubeconfig is where credentials end up on people's laptops:

- **Prefer exec plugins** over embedded certificates or tokens. A kubeconfig with no secret in it can't leak one.
- **Never commit kubeconfigs** with embedded credentials, and watch for them in CI logs and support tickets.
- **Use one context per cluster and role** (`prod-view`, `prod-admin`) so people work with low permissions by default and switch deliberately.
- **Clean up** old `admin.conf` copies from early cluster days. On kubeadm clusters, newer versions put that file in the `kubeadm:cluster-admins` group bound through RBAC, but older copies may still be in `system:masters`.

## Good practices

- **Use an identity provider for humans.** OIDC through the AuthenticationConfiguration file on self-managed clusters, cloud IAM on managed ones. Keep client certificates for bootstrap and break-glass.
- **Prefix every external username and group** (`oidc:`, `github:`) and reject the `system:` prefix with a `userValidationRules` expression.
- **Keep ID tokens short-lived** so disabling a user in the IdP takes effect within minutes.
- **Bind RBAC to groups**, manage group membership in the IdP, and keep bindings in version control.
- **Federate CI** with your CI provider's OIDC tokens instead of storing kubeconfigs, and restrict by repository and branch.
- **One service account per application**, automounting off by default, and never shared with people.
- **Restrict anonymous access** to health endpoints.
- **Treat CSR approval as an admin permission**, and keep any certificates you issue short-lived.
- **Review access regularly** from the RBAC side and compare it with the IdP; remove orphaned bindings.
- **Keep one break-glass credential** outside `system:masters`, stored offline, and rotate it after use.

## Bottom line

- **Kubernetes doesn't store human users.** It trusts an external system (a CA, an IdP or cloud IAM) to say who someone is, and stores only the RBAC that says what they may do.
- **Every identity is a username, a uid, groups and extras.** Bind permissions to groups and manage membership where the people are managed.
- **Client certificates are easy but can't be revoked.** Fine for bootstrap and emergencies, poor for a team.
- **Structured authentication configuration** (stable since v1.34) makes OIDC a first-class option: multiple issuers, CEL claim mapping and validation, reload without restart, and fine-grained control of anonymous access.
- **The same mechanism removes CI secrets**: trust the CI provider's tokens and restrict by repository and branch.
- **Offboarding speed depends on the credential type.** With an IdP and short-lived tokens, removing a person is one action in one system.

## References

[^1]: [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), Kubernetes documentation: users in Kubernetes, authentication strategies, client certificates, structured authentication configuration, anonymous access, webhook and proxy authentication, and client-go credential plugins.
[^2]: [Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/), Kubernetes documentation: the CSR API, built-in signers and `expirationSeconds`.
[^3]: [KEP-3331: Structured Authentication Config](https://www.kubernetes.dev/resources/keps/3331/), Kubernetes Enhancement Proposal, and [Kubernetes 1.30: Structured Authentication Configuration Moves to Beta](https://kubernetes.io/blog/2024/04/25/structured-authentication-moves-to-beta/), Kubernetes blog.
[^4]: [KEP-4633: Only allow anonymous auth for configured endpoints](https://www.kubernetes.dev/resources/keps/4633/), Kubernetes Enhancement Proposal (alpha v1.31, beta v1.32, stable v1.34).
[^5]: [Cluster access management](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-access-management.html), Amazon EKS Best Practices Guide: access entries and the deprecation of the `aws-auth` ConfigMap.
[^6]: [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/), Kubernetes documentation: escalation risks, including the CSR API and `system:masters`.
[^7]: [int128/kubelogin](https://github.com/int128/kubelogin), a kubectl plugin for Kubernetes OpenID Connect authentication.
[^8]: [About security hardening with OpenID Connect](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect), GitHub Actions documentation: OIDC tokens for workflows and their claims.
[^9]: [External identity provider authentication for AKS](https://learn.microsoft.com/en-us/azure/aks/external-identity-provider-authentication-overview), Microsoft Learn: AKS structured authentication based on the upstream configuration.
