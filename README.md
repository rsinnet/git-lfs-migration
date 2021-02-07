# git-lfs-migration

[![License: FDL
1.3](https://img.shields.io/badge/License-FDL%20v1.3-blue.svg)](https://www.gnu.org/licenses/fdl-1.3)

An in-depth case study of migrating a repository to [Git
LFS](https://git-lfs.github.com/) using the [BFG Repo
Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) and
[Bash](https://www.gnu.org/software/bash/).

## Copyright Notice

Copyright &copy; 2021 Ryan Sinnet.

Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.3
or any later version published by the Free Software Foundation;
with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.
A copy of the license is included in the section entitled "GNU
Free Documentation License".

## Background

Let’s say we have a repository that has binary files all through the history.
This means you have to download every file in history just to clone the repo.
That can be painfully slow and the problem gets worse over time. [Git
LFS](https://git-lfs.github.com/) offers the ability to store binary files in a
separate LFS database (part of Git) and stores pointers to these files in each
commit. When you clone a repository that uses [Git
LFS](https://git-lfs.github.com/), you only need to fetch the pointers.

[Git LFS](https://git-lfs.github.com/) works using _smudge_ and _clean_ filters.
These filters can be configured to tell Git to apply a filters after the
standard checkout and before commit. These filters are configured via a
revisioned `.gitattributes` file which applies to all paths and subpaths of the
folder in which this file exists (just like a `.gitignore` file). See
[gitattributes](https://git-scm.com/docs/gitattributes) in the [Git Reference
Manual](https://git-scm.com/docs) and [8.2 Customizing Git - Git
Attributes](https://git-scm.com/book/en/v2/Customizing-Git-Git-Attributes) in
[Pro Git](https://git-scm.com/book/en/v2).

As you might imagine, smudge and clean filters are quite powerful and you can
probably imagine how they can be used to inject secret values like passwords at
checkout and clean them at commit to prevent storing secret data in your
repository. In this case study, however, we are interested in using these
filters for storing large files through [Git LFS](https://git-lfs.github.com/).

When you check out a commit with `git checkout`, if that commit has a
`.gitattributes` file, Git will apply a smudge filter to fetch the LFS object
from the remote into the local LFS database and then replace this pointer with
the object from the local LFS database. When you commit, Git will apply a
corresponding clean filter, storing the binary object in the local LFS database
and replacing the it with a pointer. When you push in modern Git, [Git
LFS](https://git-lfs.github.com/) will automatically invoke `git lfs push
origin` as part of the `git push origin** command. See
[git-lfs-filter-process(1)](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-filter-process.1.ronn)
in the [official man
pages](https://github.com/git-lfs/git-lfs/tree/main/docs/man).

Let’s say we want to migrate an old repository to [Git
LFS](https://git-lfs.github.com/) because we have binary objects throughout
history and it takes a long time to download. In a lot of cases, no one ever
needs the historical data so makes more sense to fetch as needed. We’ll need to
do the following:

1. Mirror the repository and fetch all existing [Git
   LFS](https://git-lfs.github.com/) objects.
1. Update the default branch to track objects via [Git
   LFS](https://git-lfs.github.com/).
1. Rewrite history, migrating objects to [Git LFS](https://git-lfs.github.com/).
1. Push everything over existing refs.

Not so bad, right?

## Step-by-step

### Setup

For this case study, we will set some shell parameters to use throughout (and
download a Git tool we’ll use later**.

**Note**: You'll have to adjust the following as necessary for your repository.


```bash
export GITHUB_USER=myuser
export GITHUB_REPO=myrepo.git

export LOCAL_WS="${HOME}/git-lfs-migration-workspace"
export BFG_JAR="${LOCAL_WS}/bfg-1.13.2.jar"
wget -O "${BFG_JAR}" "https://repo1.maven.org/maven2/com/madgag/bfg/1.13.2/${BFG_JAR##*/}"
```

In the above, we use _shell parameters_ to store values in our shell session.
See [3.4 Shell
Parameters](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameters.html)
in the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

In the above, we use _shell parameter expansion_ to strip the directory
component from the filename in the `BFG_JAR` shell parameter. See
`{parameter##word}` from [3.5.3 Shell Parameter
Expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameters.html)
in the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

We also download the [BFG Repo
Cleaner](https://rtyley.github.io/bfg-repo-cleaner/).

### Making a full back up of a repository

[Git LFS](https://git-lfs.github.com/) is great because you don’t have to fetch
the entire LFS database to clone a repository. You only need to fetch upon
checkout. This also means you might want a full copy of everything to keep a
back up or maybe because you’re traveling and won’t have internet access. In
such a case, the best bet is to clone a mirror and fetch all LFS objects:

```bash
mkdir -p "${LOCAL_WS}"
git clone --mirror "git@github.com:${GITHUB_USER}/${GITHUB_REPO}" "${LOCAL_WS}/${GITHUB_REPO}"
cd "${LOCAL_WS}/${GITHUB_REPO}"
git lfs fetch --all
tar cvzf "${HOME}/${GITHUB_REPO}.tar.gz" "${LOCAL_WS}/${GITHUB_REPO}"
```

In the above, use `git clone --mirror` to clone a mirror repository. See
[git-clone
--mirror](https://git-scm.com/docs/git-clone#Documentation/git-clone.txt---mirror)
in the [Git Reference Manual](https://git-scm.com/docs).

We use `git lfs fetch` to pull all LFS objects. See
[git-lfs-fetch(1)](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-fetch.1.ronn)
in the [official man
pages](https://github.com/git-lfs/git-lfs/tree/main/docs/man).

Now you have a bare repository just like the remote (and a tarball of it in your
home directory for extra backup). You can now clone locally and use this mirror
instead of the remote.

### Adding Git LFS tracking to the default branch

Let’s add [Git LFS](https://git-lfs.github.com/) tracking to the default branch.
That means we need a gitattributes filter as described above. [Git
LFS](https://git-lfs.github.com/) has a convenient way to set up this filter for
you. Let’s say you want to use [Git LFS](https://git-lfs.github.com/) to store
protobuf files (`.pb`) containing a TensorFlow graph and model weights. This is
a great candidate because weights completely change between training sessions
and there’s no good way to track diffs. They’re also huge. Let’s say we also
want to revision STL files (`.stl`). These files are object meshes which can get
very large. We can set up the appropriate clean and smudge filters with a simple
command: `git lfs track`. We’ll do this bit of work using our mirror:

**Note**: You may need to change this depending on your workflow. Do you use
merging? Then you have to check out a new branch after cloning. Do you need a
pull request? Then you have to use the GitHub remote instead of the mirror.

```bash
git clone "${LOCAL_WS}/${GITHUB_REPO}" "${LOCAL_WS}/${GITHUB_REPO%.git}"
cd "${GITHUB_REPO%.git}"

# If you use a merge workflow, you will need to check out a new branch here
# before proceeding.
git lfs track *.{pb,stl}
git stage .gitattributes

# Stage all current .pb and .stl files.
find . -regextype posix-egrep -regex ".*\.(pb|stl)$" -exec git stage {} \;

git commit -F- <<EOF
Add .pb and .stl files to Git LFS

Track .pb and .stl files with Git LFS and migrate existing files to Git LFS.
EOF

git push origin
```

In the above, we use _shell parameter expansion_ to remove the `.git` extension
from filename in the `GITHUB_REPO` shell parameter. See `{parameter%word}` from
[3.5.3 Shell Parameter
Expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html#Shell-Parameter-Expansion)
in the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

We use a _here document_ for the commit message. See [3.6.6 Here
Documents](https://www.gnu.org/software/bash/manual/bash.html#Here-Documents) in
the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

We use `git lfs track` to add [Git LFS](https://git-lfs.github.com/) clean and
smudge filters to the `.gitattributes` file. See
[git-lfs-track(1)](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-track.1.ronn)
and
[git-lfs-untrack(1)](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-untrack.1.ronn)
in the [official man
pages](https://github.com/git-lfs/git-lfs/tree/main/docs/man).

We use the `find` command to stage files. See [2.2.2 Full Name
Patterns](https://www.gnu.org/software/findutils/manual/html_mono/find.html#Full-Name-Patterns),
[3.3 Run
Commands](https://www.gnu.org/software/findutils/manual/html_mono/find.html#Run-Commands)
and [8.5.8: ‘posix-egrep’ regular expression
syntax](https://www.gnu.org/software/findutils/manual/html_mono/find.html#posix_002degrep-regular-expression-syntax)
in the [GNU Findutils
Manual](https://www.gnu.org/software/findutils/manual/html_mono/find.html).

### Migrating to Git LFS

Now that we have [Git LFS](https://git-lfs.github.com/) set up on the default
branch, we can go through and rewrite everything else. The easiest and fastest
way to do this is with the [BFG Repo
Cleaner](https://rtyley.github.io/bfg-repo-cleaner/):

```bash
cd "${LOCAL_WS}/${GITHUB_REPO}"

# Rewrite history.
java -jar "${BFG_JAR}" --convert-to-git-lfs "*.{pb,stl}"

# Clean up orphaned data in your local Git database.
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

In the above, we use brace expansion to specify multiple filename arguments
conveniently. See [3.5.1 Brace
Expansion](https://www.gnu.org/software/bash/manual/html_node/Brace-Expansion.html)
in the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

We use `git reflog expire` to prune old reflog entries. We use `git gc` to
perform clean-up after expiring old reflog entries. See [git-reflog
--expire=<time>](https://git-scm.com/docs/git-reflog#Documentation/git-reflog.txt---expirelttimegt)
and [git-gc
--agressive](https://git-scm.com/docs/git-gc#Documentation/git-gc.txt---aggressive)
in the [Git Reference Manual](https://git-scm.com/docs).

Your mirror is now pretty much what you want to have on the remote. All
necessary refs have been updated.

### Pushing to Git LFS

This is the destructive part where you overwrite all refs on the origin. Make
sure you understand what you’re doing and that you have a proper back up before
doing anything here.

Now that you’ve migrated your mirror to [Git LFS](https://git-lfs.github.com/)
and updated all references, you’ll want to update your original remote (using
GitHub in this case study). Before doing this, making sure you don’t have any
branch protections enabled that prohibit force pushing. By default `git push
--mirror` only pushes local tracking branches, so we need to first create a
local tracking branch for every remote ref before force pushing:

```bash
cd "${LOCAL_WS}/${GITHUB_REPO}"

# Create local branches to track all refs from refs/remotes/origin/*.
for ref in $(git branch -a | grep remotes | grep -v HEAD); do
    git branch "${ref#remotes/origin/}" "${ref}"
done
```

In the above, we use command substitution `$()` to get a list of all the remote
refs. See [3.5.4 Command
Substitution](https://www.gnu.org/software/bash/manual/bash.html#Command-Substitution)
in the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

We use `git branch -a` to show all branches. See [git-branch
-a](https://git-scm.com/docs/git-branch#Documentation/git-branch.txt--a) from
the [Git Reference Manual](https://git-scm.com/docs).

We use _shell parameter expansion_ to strip the prefix from a shell parameter.
See `{parameter#word}` from [3.5.3 Shell Parameter
Expansion](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion)
in the [Bash Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

We also use `grep -v` to invert matches. See [2.1.2 Matching
Control](https://www.gnu.org/software/grep/manual/grep.html#Matching-Control) in
the [GNU Grep Manual](https://www.gnu.org/software/grep/manual/grep.html).

Now, we want to push everything. Let’s say this is a big repository with many
commits. In such a case, it may seem like Git is hanging, because Git really
doesn’t show much output during fetch and push operations. Maybe you don’t want
to wait forever with no information just hoping it’s working. Let’s use two
options to increase verbosity:

```bash
cd "${LOCAL_WS}/${GITHUB_REPO}"

GIT_TRACE=1 GIT_CURL_VERBOSE=1 git lfs push --all origin
GIT_TRACE=1 GIT_CURL_VERBOSE=1 git push --mirror --force origin
```

In the above, we use `GIT_TRACE` to show general traces. We use
`GIT_CURL_VERBOSE` to show all messages generated by `libcurl` (similar to `curl
-v`). There are many options available for controlling output. See [10.8 Git
Internals - Environment
Variables](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)
in [Pro Git](https://git-scm.com/book/en/v2).

We use `git lfs push --all` to push all LFS objects to the remote. See
[git-lfs-push(1)](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-push.1.ronn)
in the [official man
pages](https://github.com/git-lfs/git-lfs/tree/main/docs/man).

We use `git push --mirror` to mirror the local repository on the remote. See
[git-push --mirror](https://git-scm.com/docs/git-push) in the [Git Reference
Manual](https://www.gnu.org/software/bash/manual/html_node/index.html).

## Wrap-up

That’s it! You’ve now migrated to [Git LFS](https://git-lfs.github.com/) and
updated all refs on the remote! The `HEAD` has a `.gitattributes` file so [Git
LFS](https://git-lfs.github.com/) acts automatically.

### Checking out old commits

If you check out old commits from before you added the `.gitattributes` file,
you may need to manually apply the [Git LFS](https://git-lfs.github.com/) smudge
filter. Are your tracked files there and are they the correct file size? Or are
the files just filled with pointers to the [Git
LFS](https://git-lfs.github.com/) database? If they're pointers, you can resolve
them:

```bash
git lfs checkout
```

See
[git-lfs-checkout(1)](https://github.com/git-lfs/git-lfs/blob/main/docs/man/git-lfs-checkout.1.ronn)
in the [official man
pages](https://github.com/git-lfs/git-lfs/tree/main/docs/man).

### Force-pushing in a team environment

Force-pushing refs on a shared remote may cause a lot of strife if you have many
users of the repo. If a user has been working on a local clone of the repository
and you’ve force-pushed over existing refs, the user's local branches will still
be based on the old refs. The user should be able to simply fetch the updated
refs and then rebase onto the latest default branch. Assuming the user is in the
repository directory and has the branch of interest checked out:

```bash
# Determine the default branch
DEFAULT_BRANCH="$(git remote show origin | grep "HEAD branch" | cut -d ":" -f 2 | sed -e 's/^[[:space:]]*//')"
git fetch --all origin
git rebase --interactive --autosquash origin/develop
git push --force-with-lease origin
```

**Note**: These commands are not part of the migration and would be done by a
user in the user’s local repository clone.

In the above, we use `cut` to parse the output into fields based on the
specified `:` delimiter and then print only the second field. See
[cut(1)](https://linux.die.net/man/1/cut) from the Linux man pages and [8.1 cut:
Print selected parts of
lines](https://www.gnu.org/software/coreutils/manual/coreutils.html#cut-invocation)
from the [GNU Coreutils
Manual](https://www.gnu.org/software/coreutils/manual/coreutils.html).

We use `sed -e` and the `s` command. We also use a _bracket expression_. See
[2.2 Command-Line
Options](https://www.gnu.org/software/sed/manual/sed.html#Command_002dLine-Options),
[3.3 The s
Command](https://www.gnu.org/software/sed/manual/sed.html#The-_0022s_0022-Command),
and [5.5 Character Classes and Bracket
Expressions](https://www.gnu.org/software/sed/manual/sed.html#Character-Classes-and-Bracket-Expressions)
from the [GNU sed Manual](https://www.gnu.org/software/sed/manual/sed.html).

If the user wants to then clean out all old data, the user could do the
following:

```bash
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

We use `git reflog expire` to prune old reflog entries. We use `git gc` to
perform clean-up after expiring old reflog entries. See [git-reflog
--expire=<time>](https://git-scm.com/docs/git-reflog#Documentation/git-reflog.txt---expirelttimegt)
and [git-gc
--agressive](https://git-scm.com/docs/git-gc#Documentation/git-gc.txt---aggressive)
in the [Git Reference Manual](https://git-scm.com/docs).

### GitHub maintains hidden refs to pull requests

GitHub maintains read-only refs to previous pull requets under
`/refs/pulls/ID/head` where the `ID` refers to the pull request ID (pull request
24 on GitHub). Even if you rewrite and force-push history, these refs will still
exist, because you will be unable to force-push over them. That means the old
tree still exists on GitHub in some form. That’s fine because by default, these
refs are not fetched when a user clones the repository. They can be fetched if
desired. To fetch and check out pull request 24, for example, you can do the
following:

```bash
LOCAL_BRANCH=
git fetch origin "pull/24/head:${LOCAL_BRANCH}"
git checkout "${LOCAL_BRANCH}"
```

See [Checking out pull requests locally](Checking) from the [GitHub
Docs](https://docs.github.com/en).

**Note**: Fetching may take a while because you have to download tree of old
commits which has the unmigrated data stored without [Git
LFS](https://git-lfs.github.com/). These refs aren’t fetched by default when you
do `git clone`, so you won't have massive downloads with a normal `git clone`
operation.
