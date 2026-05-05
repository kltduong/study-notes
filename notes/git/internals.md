# Git Internals: A Concise Reference

**TL;DR** — Git is a content-addressable key-value store keyed by SHA-1. Commits point to trees, trees point to blobs and subtrees, branches are just files in `refs/heads/` holding a commit hash. Config resolves across system → global → local, with `includeIf` for path-conditional overrides.

## Git Configuration Files

Git reads configuration from three levels, with more specific levels overriding more general ones:

**System config** (`/etc/gitconfig`) applies to every user on the machine. Rarely edited directly; modified via `git config --system`.

**Global config** (`~/.gitconfig` or `~/.config/git/config`) applies to your user across all repositories. This is where you typically set your identity, default editor, aliases, and preferred behaviors. Edit via `git config --global <key> <value>` or open the file directly.

**Local config** (`.git/config` inside a repo) applies only to the current repository and overrides global settings. This is where remotes, branch tracking info, and per-project overrides live. Edit via `git config --local <key> <value>` (the default when no flag is given).

To inspect settings and see where they came from: `git config --list --show-origin`.

A typical global config looks like:

```ini
[user]
    name = Your Name
    email = you@example.com
[core]
    editor = vim
[alias]
    co = checkout
    st = status
```

## Handling Multiple Git Accounts

A common scenario is needing different identities (and SSH keys) for work vs. personal repositories. There are three clean approaches.

### 1. Per-Repo Override

The simplest fix: override `user.name` and `user.email` inside a specific repo.

```bash
cd ~/projects/work-repo
git config --local user.email "you@company.com"
```

This only changes who the commits are attributed to — not which SSH key is used for pushing.

### 2. Conditional Includes (Recommended)

Git supports conditionally loading a config file based on the repository's path. Set this up once in your global config and it applies automatically to any repo in the matching directory.

```ini
# ~/.gitconfig
[user]
    name = Your Name
    email = personal@example.com

[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
```

Then create `~/.gitconfig-work`:

```ini
[user]
    email = you@company.com
[core]
    sshCommand = "ssh -i ~/.ssh/id_work"
```

Any repo cloned under `~/work/` now automatically uses the work email and work SSH key. No per-repo setup needed.

> Gotcha: the trailing slash in `gitdir:~/work/` matters — without it, the pattern matches differently. Also, `includeIf` is evaluated when git reads config, so the directory must already exist at the matched path.

### 3. SSH Config Host Aliases

For managing multiple SSH keys (e.g., two GitHub accounts), define host aliases in `~/.ssh/config`:

```
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_personal

Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_work
```

Then clone using the alias instead of the real hostname:

```bash
git clone git@github.com-work:company/repo.git
```

Git routes through the alias, which picks the right key. Combine this with conditional includes for a fully automated setup.

---

## The `.git` Directory Structure

Everything Git knows about your repository lives inside `.git/`. The most important pieces:

- **`HEAD`** — a single file that points to the currently checked-out branch. Usually contains `ref: refs/heads/main`. In a detached HEAD state, it holds a commit hash directly instead.
- **`refs/`** — human-readable pointers to commits. `refs/heads/` holds branches, `refs/tags/` holds tags, `refs/remotes/` tracks remote branches. Each file is tiny: just the 40-character SHA-1 hash of the commit it points to.
- **`objects/`** — Git's database. Every piece of content Git tracks (files, directories, commits) is stored here as a compressed object, addressed by its hash.
- **`config`** — the local config file discussed above.

## Hashing Functions (Background)

A hash function maps any input to a fixed-size output. A good cryptographic hash is deterministic, fast to compute, one-way, and collision-resistant. Git uses SHA-1 by default, producing a 40-character hex string. (Newer Git versions also support SHA-256 repos via `git init --object-format=sha256`, but SHA-1 remains the default.) This property is the foundation of Git's entire storage model.

## Git as a Key-Value Store

At its core, Git is a content-addressable key-value database. The **key** is the SHA-1 hash of the content, and the **value** is the content itself (compressed). Identical content is only ever stored once, and any change to content produces a completely different hash — giving Git its integrity guarantees essentially for free.

## Writing and Reading Objects

- **`git hash-object`** computes the SHA-1 of given content. With `-w`, it also writes the object into `.git/objects/`. Git stores objects by splitting the hash: the first 2 characters become a subdirectory, the remaining 38 become the filename.
- **`git cat-file`** retrieves objects. Common flags: `-t` shows type, `-p` pretty-prints content, `-s` shows size.

## The Object Types

Git has four object types. The first three are the ones you encounter constantly:

**Blobs** store file contents — and only the contents. A blob has no filename, no permissions, no metadata. Two files with identical content share a single blob, regardless of where they live or what they're called.

**Trees** represent directories. A tree is a list of entries, each containing a mode (permissions), a type (blob or tree), a hash, and a name. Trees are what give blobs their filenames and directory structure. A tree can reference blobs (files) and other trees (subdirectories), forming the snapshot of a project at a point in time.

**Commits** tie everything together. A commit contains a reference to a single tree (the root snapshot), zero or more parent commit hashes (zero for the initial commit, one for a normal commit, two or more for a merge), author and committer info with timestamps, and the commit message. Because a commit references its parents by hash, and each parent its own tree and parents, the entire history is a tamper-evident chain — changing anything anywhere would change every hash from that point forward.

**Tag objects** (fourth type) back *annotated* tags only. They store a target hash, a tagger, a date, and a message — essentially a named, signed pointer. Lightweight tags, by contrast, are just refs under `refs/tags/` pointing straight at a commit with no tag object at all.

## The Object Graph

A commit transitively identifies every file in the snapshot through a chain of hash pointers. A commit holds *one* tree hash (the root); that tree lists blobs (files) and subtrees (subdirectories); subtrees recurse the same way until every leaf is a blob.

For a repo containing `README.md` and `src/main.py`:

```
commit abc123
  └── tree def456           (root directory)
        ├── blob 111aaa     README.md
        └── tree 222bbb     src/
              └── blob 333ccc  main.py
```

"Points to" means "stores the hash of". Changing any blob changes its hash, which changes every tree above it, which changes the commit hash — this is what makes history tamper-evident.

## Putting It Together

When you run `git commit`, Git:

1. Writes blob objects for any changed files.
2. Writes tree objects describing the directory structure.
3. Writes a commit object pointing to the root tree and the previous commit.
4. Updates the ref in `refs/heads/<branch>` to the new commit hash.

`HEAD` still points to the branch, so it now transitively points to the new commit. That's the whole model.
