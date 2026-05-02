# Project 1 — `podpeek`: A Kubectl-style CLI in Go

**Estimated time:** 4–8 hours spread over a few sittings.
**Prerequisites:** Go installed (1.22+), `minikube` running, `kubectl` configured.

> **Mentor note:** Don't speed-run this. Project 1 is where Go fluency is built. Every concept here — package layout, contexts, errors, channels — comes back in every other project. Type the code yourself, don't paste.

---

## What you're building

A command-line tool called `podpeek` that talks to a Kubernetes cluster from Go using the official `client-go` library.

```bash
$ podpeek -n kube-system
NAME                               PHASE     NODE           AGE
coredns-5d78c9869d-abcde           Running   minikube       2d
etcd-minikube                      Running   minikube       2d
kube-apiserver-minikube            Running   minikube       2d
...

$ podpeek -n default --watch
[ADDED]    nginx-deploy-xyz       Running
[MODIFIED] nginx-deploy-xyz       Running
[DELETED]  nginx-deploy-xyz       -
```

No CRDs. No controllers. Just learning to *read from* a cluster the way an operator does.

---

## Why this project comes first

An operator is just a Go program that watches Kubernetes objects and reacts to them. Before you can write one, you need three muscles:

1. **Loading a kubeconfig from Go** — operators do this on startup.
2. **Calling the Kubernetes API from Go** — every operator does this on every reconcile.
3. **Watching for events** — operators are *reactive*; they subscribe to a stream of changes rather than polling.

Project 1 builds those three muscles in isolation, without the extra weight of CRDs, RBAC, or controller-runtime. When you get to Project 2, the kubebuilder scaffolding will hide a lot of magic — but you'll already know what it's doing under the hood because you wrote the simpler version yourself.

---

## Concepts you'll consciously practice

### Go concepts

| Concept | Where it shows up | Why it matters later |
|---|---|---|
| `go mod init` and module paths | Project setup | Every Go project starts here |
| Package layout (`cmd/`, `internal/`, `pkg/`) | File structure | Operators follow the same convention |
| Structs and methods | Wrapping the K8s client | Reconcilers are just structs with a `Reconcile` method |
| Pointer vs value receivers | Methods on the client wrapper | Reconcilers always use pointer receivers |
| `context.Context` | Every API call | Required for cancellation in operators |
| Error wrapping with `fmt.Errorf("...: %w", err)` | Every error path | Operator logs are useless without this |
| Channels and `select` | The `--watch` flag | Same pattern used inside controller-runtime |
| `cobra` for CLI flags | Argument parsing | `kubectl` itself is built with cobra |

### Kubernetes concepts

| Concept | What you'll learn |
|---|---|
| kubeconfig | The file at `~/.kube/config` and how Go programs load it |
| `Clientset` (typed client) | The "easy mode" Kubernetes client — strong types, autocomplete |
| Dynamic client | The "hard mode" client — used when you don't know the type at compile time |
| `Pod` object structure | `Spec`, `Status`, `Metadata` — the shape every K8s object follows |
| Watch streams | How K8s pushes change events to clients |
| Label selectors | The standard query language across K8s |

---

## Step-by-step build guide

### Step 0 — Verify your environment

```bash
go version           # Should be 1.22 or newer
minikube status      # Should say "host: Running"
kubectl get nodes    # Should list "minikube"
```

If any of these fail, fix that first. Don't proceed with a broken environment.

### Step 1 — Initialize the Go module

```bash
mkdir -p ~/code/podpeek && cd ~/code/podpeek
go mod init github.com/<your-username>/podpeek
```

**What just happened:** `go mod init` created a `go.mod` file. This is Go's equivalent of `package.json` or `requirements.txt`. The module path (`github.com/...`) is how *other* code would import yours; for now, it's just a unique identifier.

> **Concept check:** Open `go.mod` and read it. The `module` line is the import path. The `go` line declares the minimum Go version. Dependencies will be added to a `require` block automatically as you import them.

### Step 2 — Plan the package layout

Create this structure:

```
podpeek/
├── go.mod
├── main.go                     # entrypoint, just calls cmd.Execute()
├── cmd/
│   ├── root.go                 # cobra root command + global flags
│   ├── list.go                 # the default "list pods" command
│   └── watch.go                # the --watch behavior
└── internal/
    └── kube/
        └── client.go           # wraps clientset creation
```

**Why this layout:**
- `cmd/` holds the CLI surface — one file per command keeps things readable.
- `internal/` is a Go convention: packages here can only be imported by *this* module, not by anyone who imports yours. It's how you mark "private" code.
- `internal/kube/` is where K8s-specific code lives. In Project 2 you'll see that real operators put their controllers in `internal/controller/` for the same reason.

> **Mentor pause:** Before continuing, create the empty files. Run `tree` or `ls -R` and confirm the layout matches.

### Step 3 — Add the dependencies

```bash
go get k8s.io/client-go@latest
go get k8s.io/apimachinery@latest
go get github.com/spf13/cobra@latest
```

**What just happened:** `go get` downloaded these modules and added them to `go.mod`. A new file, `go.sum`, was also created — it pins exact versions and checksums for reproducible builds. **Commit both files to git.**

> **Why client-go and apimachinery are separate:** `client-go` is the *client* (it makes HTTP requests). `apimachinery` defines the *types* (`Pod`, `Service`, `ObjectMeta`, etc.). Other projects also reuse the types without needing the client, so they're split. You'll import from both constantly.

### Step 4 — Write `internal/kube/client.go`

This file's job: load the kubeconfig and return a ready-to-use `Clientset`.

**Your turn — try this yourself first.** Look up these symbols in the Go docs:
- `clientcmd.BuildConfigFromFlags`
- `kubernetes.NewForConfig`
- `homedir.HomeDir` (from `k8s.io/client-go/util/homedir`)

The function signature should be:

```go
func NewClientset() (*kubernetes.Clientset, error)
```

It should:
1. Find the kubeconfig path (default to `~/.kube/config`, allow override via `KUBECONFIG` env var).
2. Build a `*rest.Config` from it.
3. Create and return a `*kubernetes.Clientset`.
4. Wrap any error with context: `fmt.Errorf("loading kubeconfig: %w", err)`.

**If you get stuck for more than 20 minutes**, look at the reference solution at the bottom of this file.

> **Concept check:** Why return `*kubernetes.Clientset` (a pointer) rather than the value? Because `Clientset` is large and contains internal state — copying it would be wasteful and would lose shared connection pools. As a rule in Go: structs that hold resources (clients, files, locks) are passed by pointer.

### Step 5 — Write `cmd/root.go` (cobra setup)

`cobra` is the CLI library used by `kubectl`, `helm`, `gh`, and most modern Go CLIs. Patterns to learn:

- A `*cobra.Command` represents one command (or subcommand).
- `RunE` is the function that runs when the command is invoked. The `E` suffix means "returns an error" — *always prefer `RunE` over `Run`*.
- Persistent flags are inherited by subcommands; local flags aren't.

Add a persistent `--namespace` (`-n`) flag that defaults to `"default"`, and a persistent `--kubeconfig` flag.

### Step 6 — Implement the list command

In `cmd/list.go`, write a function that:

1. Calls `kube.NewClientset()`.
2. Creates a `context.Context` with a 30-second timeout: `ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second); defer cancel()`.
3. Calls `clientset.CoreV1().Pods(namespace).List(ctx, metav1.ListOptions{})`.
4. Iterates over `podList.Items` and prints each pod's name, phase, node name, and age.
5. Use `text/tabwriter` from the standard library for nicely-aligned output.

**Computing age:** `time.Since(pod.CreationTimestamp.Time)` gives a `time.Duration`. Format it as `2d`, `5h`, `12m` — write a small helper.

> **Concept check:** Why pass `ctx` to `List`? Because if the user hits Ctrl-C, you want the in-flight HTTP request to be cancelled too. Every K8s client method takes a context for exactly this reason. **Operators use the same pattern** — every reconcile receives a context, and if the operator is shutting down, that context is cancelled and your in-flight calls abort cleanly.

### Step 7 — Implement the `--watch` flag

This is where channels enter the picture.

In `cmd/watch.go`:

```go
watcher, err := clientset.CoreV1().Pods(namespace).Watch(ctx, metav1.ListOptions{})
if err != nil {
    return fmt.Errorf("starting watch: %w", err)
}
defer watcher.Stop()

for event := range watcher.ResultChan() {
    pod, ok := event.Object.(*corev1.Pod)
    if !ok {
        continue
    }
    fmt.Printf("[%s]\t%s\t%s\n", event.Type, pod.Name, pod.Status.Phase)
}
```

**Things to notice and understand:**

- `watcher.ResultChan()` returns a `<-chan watch.Event` — a *receive-only* channel. You can read from it but not send to it. Read-only channels are how Go expresses "this is a stream you consume."
- The `for ... range` loop blocks waiting for events. It exits when the channel closes (on watcher stop or context cancellation).
- The type assertion `event.Object.(*corev1.Pod)` is needed because watch events carry a generic `runtime.Object`. This is one of the few places Go feels weakly typed — and it's exactly why the *typed* clientset exists for the common cases.

> **Concept check:** What happens if the network blips and the watch connection drops? The channel closes and your loop exits silently. Real operators wrap this in a *reflector* that auto-reconnects with the last seen `resourceVersion`. We're not handling that here — but knowing it's a problem is part of becoming an expert. Project 2's controller-runtime handles it for you.

### Step 8 — Wire it all together in `main.go`

```go
package main

import (
    "fmt"
    "os"

    "github.com/<you>/podpeek/cmd"
)

func main() {
    if err := cmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

> **Concept check:** Why print to `os.Stderr` instead of `os.Stdout`? Because errors aren't part of the program's *output* — they're diagnostics. A user piping `podpeek | grep nginx` shouldn't see error messages mixed into the grep input.

### Step 9 — Build and run

```bash
go build -o podpeek
./podpeek -n kube-system
./podpeek -n default --watch
```

Open a second terminal and `kubectl run test --image=nginx` — you should see ADDED/MODIFIED events stream in the first terminal. Then `kubectl delete pod test` — DELETED events appear.

### Step 10 — Polish

```bash
gofmt -w .       # formats your code
go vet ./...     # static analysis
```

Both should produce zero output. **Don't skip this.** `gofmt` is non-negotiable in Go culture; `go vet` catches real bugs.

---

## Acceptance criteria

Tick each box before moving to Project 2:

- [ ] `minikube start` is running
- [ ] `podpeek -n kube-system` lists pods in a clean table
- [ ] `podpeek -n default --watch` streams ADDED/MODIFIED/DELETED events live
- [ ] Cancelling with Ctrl-C exits cleanly (no goroutine panic, no stack trace)
- [ ] `go vet ./...` is clean
- [ ] `gofmt -l .` returns nothing (everything is formatted)
- [ ] `go.mod` and `go.sum` are committed to git

## Mentor checkpoint questions

When you finish, be ready to answer (out loud or in writing):

1. **Why do we pass `context.Context` to every K8s API call?** What would happen without it?
2. **What's the difference between `clientset.CoreV1().Pods(ns).List()` and the dynamic client?** When would you reach for the dynamic one?
3. **Walk me through what happens when the watch connection drops mid-stream.** What signal does your code receive? What would a *production* tool do differently?
4. **Why is `internal/` named that way?** What does Go enforce about it?
5. **In `for event := range watcher.ResultChan()`, when does the loop exit?**

If any of these feel hand-wavy, go back and re-read that part of your code.

---

## Stretch goals

Worth doing if you have an extra evening:

1. **`--selector` flag.** Accept a label selector string (e.g. `app=nginx,tier=frontend`) and pass it via `metav1.ListOptions{LabelSelector: ...}`.

2. **`--output json` / `--output yaml` flag.** Marshal the pod list using `encoding/json` or `sigs.k8s.io/yaml`. This teaches you about Go struct tags (`json:"name"`).

3. **Subcommand for services.** `podpeek services -n default`. This forces you to refactor the client wrapper to be reusable across resource types.

4. **A test.** Write one unit test using the `fake` clientset from `k8s.io/client-go/kubernetes/fake`. This is the same fake used in real operator tests — and you'll use it in Project 2.

---

## Reference solutions (only after attempting yourself)

> **Don't peek until you've spent at least 30 minutes stuck.** Struggle is where learning happens.

### `internal/kube/client.go` reference

```go
package kube

import (
    "fmt"
    "os"
    "path/filepath"

    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientconfig/clientcmd"
    "k8s.io/client-go/util/homedir"
)

func NewClientset(kubeconfigPath string) (*kubernetes.Clientset, error) {
    if kubeconfigPath == "" {
        if env := os.Getenv("KUBECONFIG"); env != "" {
            kubeconfigPath = env
        } else if home := homedir.HomeDir(); home != "" {
            kubeconfigPath = filepath.Join(home, ".kube", "config")
        }
    }

    config, err := clientcmd.BuildConfigFromFlags("", kubeconfigPath)
    if err != nil {
        return nil, fmt.Errorf("loading kubeconfig from %q: %w", kubeconfigPath, err)
    }

    cs, err := kubernetes.NewForConfig(config)
    if err != nil {
        return nil, fmt.Errorf("creating clientset: %w", err)
    }

    return cs, nil
}
```

### Age-formatting helper

```go
func formatAge(t time.Time) string {
    d := time.Since(t)
    switch {
    case d < time.Minute:
        return fmt.Sprintf("%ds", int(d.Seconds()))
    case d < time.Hour:
        return fmt.Sprintf("%dm", int(d.Minutes()))
    case d < 24*time.Hour:
        return fmt.Sprintf("%dh", int(d.Hours()))
    default:
        return fmt.Sprintf("%dd", int(d.Hours()/24))
    }
}
```

(Compare yours with `kubectl`'s output — it follows the same pattern.)

---

## What changes in Project 2

You'll throw away most of this code — but every concept comes back:

- The `*kubernetes.Clientset` you built becomes `client.Client` from `controller-runtime` (a richer wrapper).
- The `context.Context` plumbing is identical.
- The watch-stream pattern is what controller-runtime hides behind its `Reconcile` callback.
- Your error-wrapping habits stay exactly the same.

When kubebuilder generates a controller skeleton, you'll recognize what it's doing because you wrote the simpler version by hand.

**Move on to Project 2 only after the acceptance criteria are all green.**
