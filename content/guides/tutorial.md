+++
title = "StGit Tutorial"
+++

StGit is a command-line application that provides functionality similar
to [Quilt](http://savannah.nongnu.org/projects/quilt) or the [Mercurial
Queues extension](https://www.mercurial-scm.org/wiki/MqExtension), i.e.
pushing and popping patches to/from a stack, but using Git instead of
`diff` and `patch`. StGit patches are stored in a Git repository as Git
commits, but can be manipulated by StGit commands in a variety of
powerful ways beyond what can easily be done with regular Git commits.

This tutorial assumes familiarity with the basics of Git, including
commits, branches, and merge conflicts. For more information on Git, see
[git(1)](https://git-scm.com/docs/git) or [the Git home
page](https://git-scm.com/).

## Getting Started

### Online Help

For a full list of StGit commands:

```
$ stg help
```

For quick help on an individual `stg` subcommand:

```
$ stg help <cmd>
```

For more extensive help on a subcommand:

```
$ man stg-<cmd>
```

The documentation is also available as [online man pages](/man).


### Setup a Repository

StGit operates in the context of a regular Git repository, providing
additional capabilities above and beyond those provided by Git.

This tutorial uses StGit's own Git repository for its examples, but
any regular Git repository may be used to work through this tutorial.

Use `git init` to create or `git clone` to clone a Git repository.

To clone the StGit repository:

```
$ git clone https://github.com/stacked-git/stgit.git
$ cd stgit
```

Before creating StGit patches, the StGit stack must be initialized
on the current Git branch using [`stg init`](/man/stg-init):

```
$ stg init
```

This initializes the StGit stack metadata for the current branch. To
have StGit patches on another branch, `stg init` must be run again
on that branch.

## Patches

### Create a Patch

With the StGit stack initialized, patches may be created:

```
$ stg new my-first-patch
```

This will create a patch called `my-first-patch`, and open an editor to
edit the patch's commit message. This patch is empty, which can be seen
by running [`stg show`](/man/stg-show):

```
$ stg show
```

> **NOTE** `stg new` may be called without a patch name, in which case
> the patch name will be generated by StGit based on the first line of
> the commit message.

But empty patches are not particularly interesting. So the next step is
to make a modification to a file in the working tree using a regular
text editor.

```
$ $EDITOR setup.py
$ stg status
 M README.md
```

To update a patch with changes from the working tree, [`stg
refresh`](/man/stg-refresh) is used:

```
$ stg refresh
```

And voilà -- the patch is no longer empty:

```
$ stg show
commit d443f7e2d1099d07b37de02ec483691521e3c330 (HEAD -> master, refs/patches/master/my-first-patch)
Author: Audrey U. Thor <author@example.com>
Date:   Tue Sep 20 13:56:33 2022 -0400

    Tis but a patch

diff --git a/README.md b/README.md
index c8a0894ac..0ef993493 100644
--- a/README.md
+++ b/README.md
@@ -81,3 +81,5 @@ to StGit.
 StGit is maintained by Catalin Marinas and Peter Grayson.
 
 For a complete list of StGit's authors, see [AUTHORS.md](AUTHORS.md).
+
+This is my first patch!
```

Since the patch is also a regular Git commit, it can be seen by regular
Git tools such as [`gitk`](https://git-scm.com/docs/gitk).

### Another Topic, Another Patch

When making a change for a new topic, create a new patch. This time the
working tree is modified *before* creating the new patch. It is a
feature of StGit that a patch can be created independent of the working
tree state.

```
$ echo '- Audrey U. Thor' >> AUTHORS.md
$ stg new credit --message 'Give me some credit'
$ stg refresh
```

> **NOTE** Use the `--message` (`-m`) option to [`stg
> new`](/man/stg-new) to give the patch a message without invoking an
> editor.

> **NOTE** Use the `--refresh` (`-r`) option to [`stg
> new`](/man/stg-new) to both create a new patch and refresh it in one
> step.

The stack now contains two patches:

```
$ stg series --description
+ my-first-patch # Tis but a patch
> credit         # Give me some credit
```
[`stg series`](/man/stg-series) lists the patches from bottom to top;
`+` means that a patch is 'applied', and `>` that it is the current, or
topmost, patch.

Further changes to the topmost patch can be made by just editing files
in the working tree and running `stg refresh` to capture those changes
in the topmost patch.

But how to change `my-first-patch`? The simplest way is to
[pop](/man/stg-pop) the `credit` patch. Doing so will make
`my-first-patch` the topmost applied patch again:

```
$ stg pop credit
- credit
> my-first-patch
$ stg series --description
> my-first-patch # Tis but a patch
- credit         # Give me some credit
```

[`stg series`](/man/stg-series) now indicates that `my-first-patch` is
topmost again. And running [`stg refresh`](/man/stg-refresh) will update
`my-first-patch` with any changes made to the working tree.

The minus sign (`-`) in front of `credit` in the `stg series` output
indicates that the `credit` patch is 'unapplied', which means that the
changes embodied in the `credit` patch are not currently applied to the
work tree. Unapplied patches are not seen in the regular Git history as
seen by `git log` or `gitk`.

An unapplied patch is reapplied and made the topmost patch using [`stg
push`](/man/stg-push):

```
$ stg push credit
> credit
```

> **NOTE** [`stg push`](/man/stg-push) and [`stg pop`](/man/stg-pop) may
> be called without specifying a patch name. Doing so causes the next
> unapplied patch to be pushed, or the topmost patch to be popped,
> respectively.

### Advanced Patch Refresh

By default `stg refresh` captures changes from the work tree into the
topmost applied patch, however when working with multiple patches it is
often the case that a change in the work tree should be captured by an
already applied patch. The `--patch` (`-p`) option may be used with `stg
refresh` to do just that.

```
$ stg series
+ my-first-patch
> credit
```

```
$ $EDITOR README.md
$ stg refresh --patch my-first-patch
> refresh-temp (new)
- credit..refresh-temp
> refresh-temp
- refresh-temp
# refresh-temp
& my-first-patch
> credit
```

```
$ stg series
+ my-first-patch
> credit
```

After the above refresh operation, the topmost patch remains `credit`,
but the changes from the work tree are now part of `my-first-patch` and
the work tree is clean as evidenced by running `stg status`.

> **NOTE** Another way to update a non-topmost patch is to create a new
> patch, refresh the new patch with work tree changes, and then combine
> the new patch with the other already applied patch using [stg
> squash](/man/stg-squash).


### About Commit Messages

An important part of StGit's value is to help create useful Git history.
And critical to Git commit history are good commit messages.

When creating a patch with [`stg new`](/man/stg-new), an initial commit
message is drafted, but as a patch evolves with refreshes and
reorderings, the commit message typically needs to evolve as well.

StGit makes it easy to modify (and re-modify) a patch's commit message
using [`stg edit`](/man/stg-edit):

```
$ stg edit
```

In addition to using `stg edit` anytime, a patch's message may also be
modified when refreshing by using `stg refresh --edit`.

> **NOTE** Using [stg edit](/man/stg-edit)'s `--diff` option causes the
> patch's diff text to be present inline when editing the commit
> message, which can be helpful aid for writing a thorough commit
> message.

> **NOTE** The commit message of any patch in the stack may be modified
> at any time using `stg edit <patchname>`. A patch **does not** need to
> be applied (pushed) in order to modify its commit message, so editing
> patches' commit messages may be done without risk of encountering a
> merge conflict.


### Renaming Patches

If a patch changes considerably, it might even deserve a new name.
Use [stg rename](/man/stg-rename) to rename a patch.


## Conflicts

Like with regular Git, there are various times in the normal use of
StGit when *conflicts* may occur. With regular Git commands, a conflict
may occur during [merge](https://git-scm.com/docs/git-merge),
[rebase](https://git-scm.com/docs/git-rebase), or
[pull](https://git-scm.com/docs/git-rebase). With StGit, conflicts may
occur when applying or reordering a patch using one of the following
commands:

- [`stg push`](/man/stg-push)
- [`stg goto`](/man/stg-goto)
- [`stg pick`](/man/stg-pick)
- [`stg float`](/man/stg-float)
- [`stg sink`](/man/stg-sink)
- [`stg pull`](/man/stg-pull)
- [`stg rebase`](/man/stg-rebase)
- [`stg refresh --patch`](/man/stg-refresh)

Normally, when re-pushing a patch after popping it and making a change
to another patch, StGit is able to re-push the patch without conflict.
In the following example, two patches are created that each modify a
different file. First a test repository is initialized:

```
$ git init test-repo
$ cd test-repo
$ touch a.txt b.txt
$ git add a.txt b.txt
$ git commit -m "Add files"
$ stg init
```

Then the two patches are created:

```
$ stg new first -m 'First patch'
$ echo 'a change' >> a.txt
$ stg refresh
$ stg new second -m 'Second patch'
$ echo 'b change' >> b.txt
$ stg refresh
```

Then both patches are popped:

```
$ stg pop --all
```

Next, the patches are reordered by pushing in the opposite order:

```
$ stg push second first
$ stg series
+ second
> first
```

StGit had no problems reordering these patches since they did not affect
the same lines or even the same files. When using StGit, it is the
typical case that no conflicts emerge when pushing or reordering
patches; especially when each patch is limited to one coherent topic.

But it is inevitable that sometimes multiple patches necessarily affect
the same lines of a file. This is when a conflict may arise.

```
$ stg pop
- first
> second
$ echo 'another change' >> a.txt
$ stg refresh
```

Now, both patches add a line to the end of `a.txt`. What happens
when attempting to apply both patches at once?

```
$ stg push
> first (conflict)
error: Merge conflicts
UU a.txt
```

StGit indicates that when it pushed `first` on top of `second` that
since both modify the same lines of the same file (`a.txt`), there is a
conflict. [`stg status`](/man/stg-status) can be used to see the status
of files in the work tree, including those with conflicts:

```
$ stg status
UU a.txt
```

As indicated by [stg push](/man/stg-push), the conflict is in the file
`a.txt`. If the patch modified multiple files, all modified files would
be listed in the `status` output, prefixed with `UU` if there were
unresolvable conflicts, or `M` if StGit was able to resolve all diff
hunks within the file.

When conflicts occur, there are two general options for how to respond:

1. Undo the command that caused the conflict(s).
2. Resolve the conflicts.

### Undo

The [`stg undo`](/man/stg-undo) command can rewind the state of the
StGit stack and work tree.

```
$ stg undo --hard
> second
```

> **NOTE** The `--hard` flag for `stg undo` is required when there are
> modifications in the work tree or index, as is the case when there
> are unresolved conflicts in the work tree.

Undoing may be helpful when inter-patch dependencies are uncovered when
attempting to reorder patches. In such cases, the best approach may be
to undo the command that caused conflicts and not reorder the patches.

In other cases, however, it may be that the conflict must be resolved
manually...

### Resolve Conflicts

Resolving conflicts incurred while using StGit commands is the same
process as when conflicts are incurred using Git commands directly:

- Modify the affected files to decide on how the conflict should be
  resolved, removing conflict markers. This can either be done manually
  using a regular text editor, or with a tool such as [`git
  mergetool`](https://git-scm.com/docs/git-mergetool)

- Tell Git that the conflict is resolved using [`git
  add`](https://git-scm.com/docs/git-add) or its StGit alias, `stg add`.

> **NOTE** It may be helpful to read the [Git Book's section on handling
> merge conflicts](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging#_basic_merge_conflicts)

Back to the example, opening `a.txt` in a text editor reveals the
following:

```
<<<<<<< current
another change
=======
a change
>>>>>>> patched
```

The 'conflict markers' `<<<<<<<`, `=======`, and `>>>>>>>` indicate
which lines were in the file before the patch was applied (`current`),
and which conflicting lines were added by the patch (`patched`).

To resolve this conflict `a.txt` is modified to choose which lines of
text to retain while also removing the conflict markers. In this case,
both lines are retained:

```
a change
another change
```

And the next step is to tell Git that the conflict has been resolved:

```
$ stg add a.txt
$ stg status
M  a.txt
```

At this point, the status indicates that `a.txt` is modified, but no
longer in conflict. The patch may now be refreshed:

```
$ stg refresh
```

The state of the resolved and refreshed patch:

```
$ stg show
commit 8e3ae5f6fa6e9a5f831353524da5e0b91727338e
Author: Audrey U. Thor <author@example.com>
Date:   Sun Oct 5 14:43:42 2008 +0200

    First patch

diff --git a/a.txt b/a.txt
index 0a131d8..b4c7416 100644
--- a/a.txt
+++ b/a.txt
@@ -1 +1,2 @@
+a change
 another change
```

## Workflows

### Development branch workflow

One common use of StGit is to "polish" a Git branch before publishing it
to another public repository. The kinds of polish that StGit can help
with include:

- Complete and correct commit messages.
- Each patch limited to one coherent topic.
- Each patch standing on its own: passing tests, etc.
- Considerate patch (commit) order

Careful curation of Git commit history, as enabled by StGit, can be
of high value to those reviewing pull requests or trying to understand
why or how code came to be the way it is.

There are limits, however, to what history may be safely modified. As a
general rule, any commits that have been made public (i.e. by pushing to
a public repository) should be off-limits to history modification.

As a concrete example, consider a situation where several Git commits
have been made in a repository with commit messages such as:

- "Improve the snarfle cache"
- "Remove debug printout"
- "New snarfle cache test"
- "Oops, spell function name correctly"
- "Fix documentation error"
- "More snarfle cache"

While the above may be the "true" history of commits to the repository,
it may not be the history that is most helpful to code reviewers or the
developer who needs to understand what happened in this are of the code
six months after the fact. Using StGit, this history can be revised to
be higher quality and higher value.

The first step is to make StGit patches from the Git commits to be
revised:

```
$ stg uncommit --number 6
> more-snarfle-cache
$ stg series --description
+ improve-the-snarfle-cache      # Improve the snarfle cache
+ remove-debug-printout          # Remove debug printout
+ new-snarfle-cache-test         # New snarfle cache test
+ oops-spell-function-name-corre # Oops, spell function name correctly
+ fix-documentation-error        # Fix documentation error
> more-snarfle-cache             # More snarfle cache
```

The [`stg uncommit`](/man/stg-uncommit) command adds StGit metadata to
the last few Git commits, turning them into StGit patches, and thus
readying them to be operated on by other StGit commands.

> **NOTE** With the `--number` flag, [`stg uncommit`](/man/stg-uncommit)
> uncommits that many commits and generates patch names based on their
> commit messages. Alternativly, patch names may be specified on the
> `stg uncommit` command line.

A number of possible history revisions are possible at this point:

- Continue developing, and take advantage of, for example [`stg
  goto`](/man/stg-goto) or `stg refresh --patch` to place modifications
  in the most appropriate patches.

- Use [`stg float`](/man/stg-float), [`stg sink`](/man/stg-sink), [`stg
  push`](/man/stg-push), and [`stg pop`](/man/stg-pop) to reorder
  patches.

- Use [`stg squash`](/man/stg-squash) to combine two or more patches
  into one. [`squash`](/man/stg-squash) pushes and pops so that the
  patches to be squashed are consecutive, then makes one big patch out
  of the patches to be squashed, and finally pushes other patches back
  on the stack such that the topmost patch is the same as it was prior
  to running `stg squash`.

> **NOTE** The above commands all cause patches to be pushed, either
> implicitly or explicitly. Thus these commands may trigger conflicts.
> If a push results in a conflict, the operation will be halted and
> the choice will have to be made to either [`undo`](/man/stg-undo) or
> resolve the conflicts.

Once the history in the StGit stack is satisfactorily revised, the
patches can be converted back into regular Git commits:

```
$ stg commit --all
```

> *TIP*: [`stg commit`](/man/stg-commit) can commit specific patches,
> leaving other patches as-is. This can be used to retire patches as
> they mature, while keeping newer and more volatile changes as patches.

When completely done using StGit with a branch, [`stg
branch`](/man/stg-branch) can be used to cleanup (remove) all StGit
metadata from the branch or completely delete the branch:

```
$ stg branch --cleanup branchname
$ stg branch --delete branchname
```

> **NOTE** A branch must have an empty stack (no patches) before it is
> either cleaned-up or deleted.


### Email-based workflow

In the 'Development branch workflow' described above, it was assumed
that only single developer was working on her own branch without having
to worry about parallel development by others. While common, this is not
the only use model for Git.

An alternative use model is for many contributors to send their patches
via email to a mailing list. This model is used, for example, by the
Linux kernel community. In this use model, others read the patches
posted to the mailing list, trying them out and providing feedback.
Often, the patch author is asked to send updated versions of patches.
When the project maintainer is satisfied with the patches, she will
apply them and publish to a public repository.

StGit is ideally suited for the process of creating patches, emailing
them out for review, revising them, mailing them off again, and
eventually getting them accepted into an upstream repository.


#### Getting patches upstream

Two StGit commands useful for sharing patches in an email-based workflow
are [`stg email`](/man/stg-email) and [`stg export`](/man/stg-export).

- [`stg email`](/man/stg-email) has two subcommands, `format` and `send`
  that may be used to format and send emails containing patches from a
  StGit stack.
- [`stg export`](/man/stg-export) exports patches from a StGit stack to
  a filesystem directory, one text file per patch. This may be useful if
  patches need to be transported by something other than email.

> **NOTE** Git has its own capability for sending commits via email:
> [`git send-email`][git-send-email]. `stg
> email send` is a wrapper of `git send-email` and as such, respects the
> its configuration (i.e. `sendemail.*` and `format.*`) and exposes its
> most used command line options. The key difference is that `stg email
> send` understands patch names from the StGit stack. Since StGit
> patches are Git commits, `git send-email` may be used directly for
> sending patches via email.

> **NOTE** For exporting a single patch [`stg show`](/man/stg-show) may
> be used instead of `stg export`.

Mailing a patch is as easy as this:

```
$ stg email send --to recipient@example.com <patches>
```

One or more patches may be listed on the command line. Each patch will
be sent as a separate email, with the first line of the commit message
used as the email's subject.

> **NOTE** [`stg email send`](/man/stg-email) relies on Git being
> properly configured to send email, e.g. via SMTP. See the
> documentation for [`git send-email`][git-send-email] and the [Git
> book's section on contributing to a project over
> email][project-over-email] for more detail on how to configure Git for
> sending email.

[git-send-email]: https://git-scm.com/docs/git-send-email
[project-over-email]: https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project#_project_over_email

There are many command-line options to control exactly how patch emails
are sent, as well as user-modifiable message templates. The [man
page](/man/stg-email) has all the details, but two worth mentioning here
are:

- `--compose` opens an editor for writing an introductory message. All
  patch emails are then sent as replies to this "cover message". Using
  a cover message is advised whenever sending more than one patch in
  order to give reviewers a quick overview of the patches.

- `--annotate` enables editing each patch before it is sent. Any part of
  the patch email may be modified, but it is not advised to edit the
  diff itself since that will only affect the outgoing email and not the
  underlying patch in the StGit stack. What `--annotate` is useful for,
  however, is to add notes for the patch recipients:

```
From: Audrey U. Thor <author@example.com>
Subject: [PATCH] First line of the commit message

The rest of the commit message

---

Everything after the line with the three dashes and
before the diff is just a comment, and not part of the
commit message. If there is anything you want the patch
recipients to see, but that should not be recorded in
the history if the patch is accepted, write it here.

 README.md |    1 +
 1 files changed, 1 insertions(+), 0 deletions(-)


diff --git a/README.md b/README.md
index e324179..6398958 100644
--- a/README.md
+++ b/README.md
@@ -171,6 +171,7 @@

+My first patch!
```


### Working with Remote Changes

In a project with multiple developers, while a local StGit stack is
being developed, others will be doing the same. As a result, a stack
will need to periodically incorporate changes from other developers.
StGit has a few different tools to help with this.

Because multiple developers may be working on the same files, pulling or
rebasing remote changes may result in conflicts. It is almost always
less work to rebase often so that smaller sets of conflicts can be
resolved versus waiting and having larger sets of conflicts to resolve.
And in most workflows, patches must be rebased prior to emailing or
creating a pull request.

#### Pulling remote changes

The most straightforward way to incorporate changes from a remote
repository is to use [`stg pull`](/man/stg-pull).

```
$ stg pull
```

Under the hood, `stg pull` first pops any applied patches, fetches
changes from a remote repository, and then reapplies any previously
applied patches. The net effect is that the patch stack is *rebased*
on top of the new remote head.

The outcome of [`stg pull`](/man/stg-pull) is thus similar to the
outcome of using [`git pull`](https://git-scm.com/docs/git-pull) with
its `--rebase` option.

> **NOTE** Pulling changes from a remote repository may result in
> conflicts when patches are reapplied. When this occurs, the `stg pull`
> command will halt after pushing the first conflicting patch. After
> resolving conflicts and refreshing the conflicting patch, it becomes
> the user's responsibility to [`push`](/man/stg-push) or
> [`goto`](/man/stg-goto) the desired patch.

> **NOTE** [`stg pull`](/man/stg-pull) pulls changes from the default
> remote associated with the branch, but a remote may also be explicitly
> specified on the `stg pull` command line.

#### Fetch remote changes and rebase

Whereas `stg pull` fetches remote changes, updates the local branch, and
rebases the branch's stack with a single command, there are times when
those steps need to be performed separately. [`stg
rebase`](/man/stg-rebase) can be used as part of a two-step process to
rebase a StGit stack with remote changes.

Step one is to fetch changes from a remote repository using either [`git
fetch`](https://git-scm.com/docs/git-fetch) or [`git remote
update`](https://git-scm.com/docs/git-remote):

```
$ git remote update
```

This updates remote branch pointers, but does not modify corresponding
local branches. Step two of this process is to update a local branch
with the remote changes and rebase the StGit stack on top of that. The
following example rebases to the *master* branch of a remote repository
"origin":

```
$ stg rebase remotes/origin/master
```

Like [`stg pull`](/man/stg-pull), [`rebase`](/man/stg-rebase) will
first pop any applied patches, update the local branch to point at
the same commit as the specified remote branch, and then reapply (push)
any previously applied patches.

The end result is that patches are now applied on top of the latest
`remotes/origin/master`.


### When patches are accepted upstream

When patches are accepted into an upstream repository, a good practice
is to pull or rebase those commits into the local repository. The one
difference in the process from above is the use of the `--merged` (`-m`)
flag with [`stg pull`](/man/stg-pull) or [`stg
rebase`](/man/stg-rebase):

```
$ stg pull --merged
```

or:

```
$ git remote update
$ stg rebase --merged remotes/origin/master
```

The `--merged` flag helps StGit detect that local patches have been
merged upstream, at some cost in performance.

The merged patches will remain present in the StGit stack after the pull
or rebase, but empty (no diff) since the change they added is now
present in the stack base. [`stg series --empty`](/man/stg-series) will
prefix any empty patches with a `0`. And [stg clean](/man/stg-clean) will
delete all empty patches from the stack:

```
$ stg series --empty
0+ patch1
 + patch2
0> patch3
$ stg clean
$ stg series --empty
 > patch2
```


### Importing patches

StGit supports importing patches from several non-Git sources using
the [`stg import`](/man/stg-import) command.

1. A patch (diff) file.

2. Several patch files containing one patch each along with a `series`
   file listing the patch files in their correct order.

3. An email containing a single patch.

4. A mailbox file (in standard Unix `mbox` format) containing
   multiple emails with one patch in each.


#### Importing a plain patch

Importing a plain patch, such as produced by e.g. GNU `diff`, `git
diff`, `git show`, [stg diff](/man/stg-diff), or [stg
show](/man/stg-show), is simply:

```
$ stg import patch-file
```

The imported patch will be at the top of the stack.

If a path is not provided on the command line, [stg
import](/man/stg-import) will read the patch from its standard input.
Thus a patch may be imported by piping the diff into `stg import`.

By default, the imported patch's name will be derived from the file
name. And if present, the patch's commit message and author info will be
taken from the beginning of the patch. However, command line options may
be used to override these defaults.


#### Importing a patch series

Some programs---among them [stg export](/man/stg-export)---will create a
directory of files with one patch per file, along with a 'series' file
(often called `series`) listing the correct patch order. Passing
`--series` with name of the series file to [stg import](/man/stg-import)
will import the entire series in its correct order.


#### Importing a patch from an e-mail

Importing a patch from an email is simple too:

```
$ stg import --mail my-mail
```

The email should be in standard Git mail format (which is what [stg
email format](/man/stg-email) produces)---that is, with the patch
in-line in the mail, not attached. The authorship info is taken from the
mail headers, and the commit message is read from the 'Subject:' header
and the mail body.

If no filename is provided, `stg import --mail` will read from stdin.
Thus a mail reader may be configured to pipe email contents into `stg
import --mail` to import (and apply) a patch email.


#### Importing a mailbox full of patches

Finally, in case importing one patch at a time is too much work, [stg
import](/man/stg-import) also accepts an entire Unix `mbox`-format
mailbox, either on the command line or on its standard input; just use
the `--mbox` flag. Each mail should contain one patch, and is imported
just like with `--mail`.

Mailboxes full of patches are produced by e.g. [stg mail](/man/stg-mail)
with the `--mbox` flag, but most mail readers can produce them too,
meaning that patch emails may be copied or moved to a separate mailbox
and then imported.
