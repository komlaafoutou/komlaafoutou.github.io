---
layout: post
title: "Git Internals, and Why Anthropic Just Walked Away From It"
date: 2026-10-13 00:00:01 -0500
categories: cs-fundamentals version-control
---

Today's news: Anthropic quietly ditched Git. In a [post describing the move](https://www.anthropic.com/engineering/redesigning-source-control), they explain that Claude writing and merging code at machine speed pushed their main branch past 100 GB, their CI jobs into 25 million checkouts a day, and their merge rate from ~2,500 to over 30,000 changes a day in five months. So they built their own source control system, called Scion, and fully migrated off Git. Git, a tool that has quietly run nearly all software on Earth for twenty years, started to buckle.

That felt like a good excuse to actually open Git up and look at how it works under the hood, slowly and from first principles, before talking about where it's starting to show its age.

## The Big Idea: Git Is a Content-Addressable Filesystem

Forget "version control" for a second. At its core, Git is a key-value store. The key is a SHA-1 (or SHA-256, more recently) hash of some content. The value is the content itself. That's it. Everything else, branches, commits, the working directory, is built on top of that one idea.

There are four object types, all stored the same way under `.git/objects`:

```
blob    → raw file contents (no filename, no metadata)
tree    → a directory listing: names + modes + pointers to blobs/trees
commit  → a pointer to one tree, plus parent(s), author, message
tag     → a pointer to a commit, with a label
```

Every object is compressed with zlib and named after the hash of its own content:

```
$ echo "hello" | git hash-object --stdin
ce013625030ba8dba906f756967f9e9ca394464

$ git cat-file -p ce013625030ba8dba906f756967f9e9ca394464
hello
```

The hash *is* the address. If two files anywhere in history have identical contents, they're the exact same blob on disk. This is why Git is cheap at storing near-duplicate files, and why changing one byte anywhere up the chain changes every hash downstream of it. Notice what's missing from a blob: no filename, no path, no permissions. A blob is pure content. Git figures out "this blob is called `main.py`" one level up, in a tree.

## Trees: Where Filenames Actually Live

A tree is just a flat list of entries. Each entry has a file mode, a name, and a pointer (hash) to either a blob (a file) or another tree (a subdirectory):

```
$ git cat-file -p 9f8e7d6
100644 blob 1111111    README.md
100644 blob 2222222    main.py
040000 tree 3333333    src
```

That's the whole mechanism for directories: a tree pointing at blobs and other trees, recursively, as deep as your folder structure goes. There's no separate "directory" concept baked into Git beyond this. `git add` and `git commit` are, underneath everything else, a process of turning your working directory into exactly this structure, bottom-up: hash every file into a blob, hash every directory's entry list into a tree, then wrap the root tree in a commit.

This is also where the famous efficiency trick lives. Say you have a repo with ten thousand files, and your next commit only changes one line in `src/utils.py`. Git has to create:

- one new blob, for the new contents of `utils.py`
- one new tree, for `src/`, pointing at the new blob (and reusing the untouched entries for anything else in `src/`)
- one new tree, for the project root, pointing at the new `src/` tree (and reusing every other entry it had before, like `README.md` and `main.py`, completely unchanged)
- one new commit, pointing at the new root tree and the previous commit as its parent

```
Before                          After
------                          -----
commit A                        commit B
 └─ tree (root)                  └─ tree (root)'          <- new
     ├─ blob README.md               ├─ blob README.md   <- reused
     ├─ blob main.py                 ├─ blob main.py     <- reused
     └─ tree src/                    └─ tree src/'        <- new
         └─ blob utils.py                └─ blob utils.py' <- new
```

Every file you *didn't* touch is reused byte-for-byte, hash-for-hash, from the previous commit. Only the path from the changed file up to the root gets new objects. That's `O(depth)` new objects per commit, not `O(repo size)`, which is the whole reason Git stores full snapshots instead of diffs without exploding in size. The "diff" you see in `git log -p` isn't stored anywhere; it's computed on the fly by walking two trees side by side and comparing what differs.

## Commits: Snapshots, Not Diffs

Put together, a commit object looks roughly like this if you print it raw:

```
$ git cat-file -p a1b2c3d
tree 9f8e7d6
parent 0a0b0c0d
author Jane Dev <jane@example.com> 1739900000 -0800
committer Jane Dev <jane@example.com> 1739900000 -0800

Fix off-by-one in retry loop
```

Five fields: a tree (the full snapshot), zero or more parents (zero for the first commit, one for a normal commit, two or more for a merge), author, committer, and a message. That's the entire data model for history. A merge commit is nothing special structurally, it's just a commit with two `parent` lines instead of one, which is how Git can later say "these two branches share this common ancestor": it walks both parent chains backward until they meet.

```
commit M (merge)
 ├─ parent: A   (tip of main)
 ├─ parent: B   (tip of feature)
 └─ tree: combined snapshot after resolving both sides
```

Walk a commit, get its tree, walk the tree recursively, write out every blob to disk: that's a full checkout, start to finish. There is no other mechanism. Every `git checkout`, `git clone`, and `git worktree add` eventually bottoms out in exactly this walk.

## Refs: Branches Are Just Pointers

With the object graph in place, everything else in Git is bookkeeping on top of it. A branch is, almost insultingly, a text file:

```
$ cat .git/refs/heads/main
a1b2c3d4e5f6...
```

One line, one commit hash. `git commit` writes a new commit object (pointing at the new tree and the old commit as parent) and then overwrites this one file with the new hash. `git branch new-feature` copies that one line into a new file. `git merge` computes a new commit with two parents and moves the current branch's file to point at it. `HEAD` is one more indirection on top: usually a file containing `ref: refs/heads/main`, so checking out a different branch is just rewriting *that* one line. ("Detached HEAD" is what happens when `HEAD` points straight at a commit hash instead of a branch name.)

It's worth sitting with how little state that actually is. The entire mutable "state" of a Git repository, which branch you're on, what each branch points to, is a handful of tiny text files. Everything else, every byte of file content and history, lives in the immutable, hash-addressed object store underneath. Mutating a branch is cheap specifically because it never has to touch any of that content, it just moves a pointer.

One more piece worth knowing, since it matters later: loose objects (one file per blob/tree/commit) get periodically repacked by `git gc` into **packfiles**, single files holding many objects delta-compressed against each other. This is what keeps a Git repo's `.git` directory from becoming an enormous pile of tiny files as history grows, and it's also the piece of Git's design that starts to hurt most at extreme scale, since repacking a packfile is a whole-repository operation.

## Why This Design Was Brilliant in 2005

Linus Torvalds built Git for exactly one workload: thousands of kernel developers, each with their own full copy of history, making mostly independent changes, syncing occasionally over email and patches. The content-addressable, fully-distributed model solves that problem elegantly:

- **Every clone is a full backup.** There's no single point of failure, because every developer has the entire history.
- **Integrity is free.** Corrupt a single byte anywhere, and the hash won't match. You find out immediately.
- **Offline-first.** Commit, branch, diff, and log all work with zero network calls. You only need the network to sync refs.
- **Branching is instant.** Since a branch is just a pointer, creating one is a single file write, not a copy of anything.

This is a beautiful design for humans who commit occasionally and need to work disconnected. It's exactly wrong for a different kind of workload.

## Where the Model Strains Today

Here's the thing Git's design assumes, bluntly: *clients are occasionally connected humans, and history is small enough to fit on a laptop.* Both of those assumptions are breaking.

**AI agents are not occasional committers.** Anthropic went from ~2,500 merges a day to over 30,000 in a few months, almost entirely driven by Claude writing and merging its own code. Each of those merges, checkouts, and diffs assumes Git's cost model, where clone is cheap-ish and local operations are instant because everything's on disk. That model doesn't hold when the "developer" is a swarm of always-on agents running millions of checkouts a day, not a human pushing a few times an afternoon.

**Full clones don't scale with monorepo growth.** Anthropic's main branch history reportedly passed 100 GB, with a working copy north of 25 GB. Git has partial clone, shallow clone, and sparse checkout bolted on to cope, but they're workarounds on a format that fundamentally wants to replicate everything, everywhere.

**Distributed-first fights server-of-truth workflows.** Most real organizations don't actually want fully distributed history. They want one authoritative server, with everyone else fetching just what they need. Scion leans all the way into that: no full clones, RPC-based checkouts that materialize only the files a job touches, and the server as the single source of truth for blame, merge, and history.

**SHA-1 and diff/blame get slower as everything grows.** These aren't fundamentally broken, but they were tuned for repos with thousands of commits and a handful of contributors, not millions of machine-generated commits a day.

None of this means Git is "wrong." It means Git optimized for a specific set of constraints, and those constraints assumed a human-paced world. When the committer is an agent that never sleeps, some of the cleverest parts of Git's design become the exact things that have to be worked around.

## Closing Thoughts

I don't think Git is going anywhere soon. Scion is Anthropic's answer to Anthropic's problem, a monorepo at a scale and commit velocity that almost no other company is at yet, and it's reportedly not being open-sourced. For the vast majority of teams, Git's object model is still the right trade: simple, verifiable, offline-friendly.

But it's a useful signal. Git's genius was designing for a world where humans write code slowly and disconnectedly. The next decade of software is going to have a lot more machine-paced, always-connected writers in the loop, and the tools we built around human cadence are going to get re-examined one by one. Source control might just be the first one people noticed.

Worth understanding the internals now, while "it's just Git" is still mostly true.
