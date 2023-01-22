# Edict: Run nix commands on remote machines
A replacement for nix remote builds that is simpler, faster, and covers more use cases.

`edict` executes `nix [build,run,flake]` commands on remote machines and copies back `./results` and other `state` changes.

## Using `edict`

---
### Get it
```command
nix shell github:davhau/edict
```
---

### **Build** remotely
```command
edict user@example.com nix build .#my-package
```
This will evaluate and build `.#my-package` remotely on `example.com`. The `./result` will appear on the local machine (together with its closure).

pass `--no-link` to not retrieve a result.

---
### **Update** your flake remotely
Any changes made on the flake by the remote are synced back to the local machine.

```command
edict user@example.com nix flake update .
```

---
### **Run** program remotely
```command
edict user@example.com nix run .#my-integration-test
```

---
### **Check** your flake remotely
```command
edict user@example.com nix flake check .
```

---
## Problems with current nix remote builds
Nix already has a remote building protocol baked in, but it has the following problems.

### 1. Non trivial authentication setup
On multi user nix installations, the nix-daemon user (root) needs passwordless ssh auth to all remote machines.

### 2. Bad performance (latency)
- Derivations are built individually requiring many round trips between client and build host.
- Outputs of each intermediary build step need to be downloaded before the next step can start.

### 3. Bad performance (bandwidth / volume)
Each intermediary build output needs to be downloaded, despite the only thing of interest being the closure of the final result.

### 4. Restricted to building
Maintaining nix infrastructure often requires more than just building derivations, like for example:
- run impure scripts to execute integration tests
- update routines which modify the repo
- enter a dev shell and execute a command

Testing these interactions on multiple systems is as important as building derivations, though the current remote build protocol lacks support for these interactions.

## The solution
The following restrictions should make it possible to build an alternative remote build command which is more powerful and less complex.

Basically what we need is just a wrapper around rsync + ssh + nix.

### 1. Build attributes, not derivations
If we stop dealing with derivations, we get rid of the complexity synchronizing individual `.drv` files and intermediary outputs between client and remote.

### 2. Eval on the build host
Evaluating the nix expression on the remote instead of the client allows us to simply let the remote do its work without having to deal with the details.

### 3. Use flakes
As we now communicate nix expressions and attribute paths, reproducible nix evaluation is very important.
We don't want the remote to derive different `.drv`'s than our client.
Flakes give us reproducible evaluation, and also makes it simpler to synchronize our nix expressions between client and remote, as it uses git to index files.

## Implementation
Create a tool that receives cmdline arguments like described at the beginning of this document and does the following:

1. Scan the arguments for flake references, like `nixpkgs#something`, or `./some/rel/path#something` or `/some/abs/path#something`.

1. Annotate each found flake reference with `local` or `remote`, depending on if it references a local directory or a remote flake.

1. Map the path of each local flake to a path on the remote

1. Rsync the local flakes to the remote.  
  (We could evaluate the store path of the local flake first to ensure we omit git ignored files, though this will cost performance on large repos)

1. `ssh` into to the remote host and execute the nix command passed to `edict` while replacing all arguments that reference a local flake with the mapped remote path.

1. Pass through stdout and stderr to the user to make it feel like the command was executed locally.

1. Scan the remote for added `result` symlinks. Create the links also on the local machine and download their closures using `nix copy`.

1. If the current directory has been passed as a flake ref, rsync back any changes that happened on it.

## Difficulties:
Is the user issues a nix command via `edict`, we don't know if they intend to modify the contents of the current directory or not.

### Examples
Here the user intends to modify the local repo:
```command
cd $HOME/my-project
edict user@example.com .#bump-versions
```
To realize this interaction we need to sync the current
directory to the remote, then execute the command and then sync back the changes.

But in the following example we definitely do not want to sync the current directory to the remote:
```command
cd $HOME
edict user@example.com nix run nixpkgs#nix-index -c nix-locate openssl.h
```

Therefore before each remote execution, we need to decide if it is a `stateful` or `stateless` interaction.

I suggest we have a heuristic that decides that, but offer the user flags like `--sync` and `--no-sync` to override the behavior.

The heuristic could work like that:  
If any of the flakes referenced in the `edict` command corresponds to the current directory, use the `stateful` mode, otherwise the `steleless` mode. 

### Future ideas
- Execute a nix command on multiple hosts in parallel
- Given a list of builders, automatically pick one matching the required `system`.
