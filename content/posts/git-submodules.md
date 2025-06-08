+++
title = 'Git Submodules'
date = 2024-05-03T23:00:55-07:00
draft = false
description = "A comprehensive guide to Git submodules, covering their implementation, common challenges, and best practices based on real-world experience from the Chromium project."
+++

Have you ever worked on a project that depends on other libraries, be that
first party or third party? Chances are, unless you are just getting started in
the tech industry, that you are. There are many ways to manage dependencies,
from utilizing dependency management systems (like pip, npm), to offloading
them to build systems (such as Bazel, Maven, etc). In some cases, it's
desirable to let the source control system handle dependencies, and this post
will cover how to do it in Git using Git submodules.

Git is the most popular DVCS used today. I'd say it has a steep learning curve
for basic stuff (pull, push, commit, branching, merging), and then steep for
learning advanced stuff (importing subtrees, rewriting history, etc). One of
those advanced stuff is working with Git submodules and understanding it. As
with many Git stuff, once you understand how things  work "under the hood",
everything starts to make sense. I believe there's a reason why the git manual
says: "git - the stupid content tracker".

## Background about my recent work
In 2023, I led a project to migrate the Chromium browser codebase from a
bespoke dependency solution to Git submodules. The transition was not smooth -
we discovered bugs throughout our codebase, but also Git
([discussion](https://lore.kernel.org/git/20230829005606.136615-1-jonathantanmy@google.com/);
both are now fully fixed). Moreover, we introduced a completely new user experience
and users weren't familiar with it.

## What are Git submodules?
Git submodules are Git's way to include other Git projects into another
project. For example, if you depend on a library called "foo", you could
import foo as a submodule. The submodule has its history, and it's completely
unaware of projects that embed them (those are often called superprojects). To
initialize, add, and inspect submodules, users can use git-submodule. `man
git-submodule` covers that in great detail. This post will focus more on
low-level details and how Git tracks submodules.

To have a proper Git submodule, you need an entry in the `.gitmodules` file,
and a gitlink - a special entry in the Git database that contains information
about the commit itself. Let's see an example of the source code of my website:

```
$ cat .gitmodules
[submodule "themes/paper"]
        path = themes/paper
        url = https://github.com/sokac/hugo-paper

$ git ls-tree HEAD -- themes/paper
160000 commit 7c94235bf3e597d0d6cc61a4261123eb7dea441c  themes/paper
```

Those two pieces of information are sufficient for Git to know how to include
themes/paper submodule into my website project (or superproject, since it has
submodules). Note that a submodule can have more information (e.g. branch
name), but those three bits of information are required.

If you enter a submodule, you can run the same Git commands as you can in a
regular Git repository. For example, you can checkout to a different revision,
commit new changes, etc. 

From a superproject perspective, there are two important states that a
submodule can be: in-sync or off-sync (e.g. git status shows `new commits`).
Moreover, Git, by default, reports some additional information about
submodules: untracked files and modified files (but not committed).

In-sync state means that a git submodule is checked out to a commit hash that
superproject expects it (again, this is stored as a gitlink). Off-sync state
means the opposite. To move from off-sync to in-sync state, the user can either
use the git submodule update operation, or manually checkout to the desired
commit hash.

## Getting started with submodules
If you have a git repository to which you want to add a new submodule, the
easiest way to do so is by using the `git submodule` command. Let's say you
have a repository https://example.org/repository.git (to clone, you would use
git clone https://example.org/repository.git). To add that as a submodule to
the existing repository, you can run: `git submodule add
https://example.org/repository.git path_you_want`. That command will perform
necessary changes to the Git database (that is, Git index) and .gitmodules
files, as well as clone the repository.

If you cd into that directory, it behaves as a regular git repository. However,
git repositories that are submodules have different .git/ paths from regular
clones - the actual content of .git/ is stored inside the superproject git
repository.

Contributing code to a submodule is straightforward - the same as contributing
to any other repository. E.g. you can checkout different branches and commit
code.

## Working on a superproject
Working on a superproject gets complicated. In the previous paragraph, I
mentioned we can contribute code to the submodule repository. Well, such
changes can be reflected in the superproject. For example, if you checkout to a
different branch (which points to a different commit), the superproject will
show an off-sync state. If you forget you did that, and use git commit -a
to commit all changes, you will include submodule change too. What is worse -
those submodule changes may only exist on your local workstation. If such a bad
commit gets pushed to remote, your teammates won't be able to sync submodules
unless the offending commit is reverted, or you push changes from the submodule
in question. And on top of that, if you use a code review system that creates a
new commit hash on submit (e.g. rebase-always strategy, or adds some metadata
to the commit message), you will be forced to revert the change in the
superproject.

I believe it's a good practice to stage files first (using git add) and commit
staged files only (i.e. don't use -a). But, that's easier said than done. With
that, let's see what other options we have when working with submodules.

## Preventing accidental submodule commits
One way to prevent accidental submodule commits was discussed above. If you
still like to stage all your changes, Git, as of this year (2024), supports
excluding submodules when adding files using pathspecs. So, instead of using
`git add -a`, you would use:

```
git add ':(exclude,attr:builtin_objectmode=160000)'
```
Since it's a long command, you should have an alias for that. See [the chromium
submodule
documentation](https://chromium.googlesource.com/chromium/src/+/main/docs/git_submodules.md#submodules-during-git-status_git-commit_and-git-add)
on how to do that.

Another way to prevent accidental commits is via git hooks. A pre-commit hook
allows you to inspect what goes into a commit. From there, a hook can instruct
Git not to move forward with the commit creation by exiting with non-zero code.
Or, the hook can unstage submodule changes (i.e. gitlinks) and continue with the
commit flow. The Chromium project uses the latter hook, and it's open-sourced
([link](https://source.chromium.org/chromium/chromium/tools/depot_tools/+/main:hooks/pre-commit.py)).
If you wish to keep it simple, you can use this simple bash script (store it in
.git/pre-commit and make sure it's executable):

```
#!/usr/bin/bash
set -ex

# Skip hook if SKIP_HOOK is 1, e.g. `SKIP_HOOK=1 git commit`
if [[ "$SKIP_HOOK" == "1" ]]; then
  exit 0
fi

diff=$(git diff-index --cached --ignore-submodules=dirty HEAD | grep "^:160000 160000")
if [[ -n "$diff" ]]; then
  echo "gitlink change detected in diff, aborting commit."
  exit 1
fi
exit 0
```

## Bonus 1: diff.ignoreSubmodules config

You can influence how some git commands will behave with
[diff.ignoreSubmodules](https://www.git-scm.com/docs/git-config#Documentation/git-config.txt-diffignoreSubmodules)
config. Although it's set in diff section, it does affect more commands rather
than just diffs.

| git command | diff.ignoreSubmodules=all | diff.ignoreSubmodules=dirty\|none |
| --- | --- | --- |
| git add .   | stages submodules | stages submodules |
| git status   | shows only staged submodules | shows staged and unstaged modified submodules |
| git commit -a | ignores submodules | commits modified submodules |
| git diff | ignores submodules | shows submodules |
| git show | ignores submodules | shows submodules |

If you are using submodules, I would personally discourage use of
diff.ignoreSubmodules=all as it hides too many things. If you are worried about
performance, setting it up to dirty is a good compromise.

*Note*: Prior to 2.42, `git commit` ignored staged git submodules with
`diff.ignoreSubmodules=all`.

## Bonus 2: Conflicts in submodules
This is probably the most confusing problem with submodules. Let's say you add
a commit that updates a submodule. And so does someone else who pushes their
code before you. You are forced to pull the latest from origin, and you get a
merge conflict. Git doesn't know what should be done and needs you to figure
that out. This can be frustrating, but let's see what is happening under the
hood.

Let's assume there was a change A which had a submodule pointing to commit1.
You created change B that points to commit2. Your teammate created change C
that points to commit3. When you rebase (or merge) your change on top of commit
3, Git sees that submodule was updated from commit 1 to commit 2 in your
commit, and from commit 1 to commit 3. So, which one is valid? Git certainly
doesn't know that.

When this happens, git won't do anything with the affected submodule. If you
just use `git add submodule_path`, it will use whatever commit hash is
currently checked out locally. This may or may not be what you want. In most
cases, you want to enter the submodule, checkout to the correct branch or
commit hash, go back to the superproject and `git add submodule_path`. If you
notice a frequent problem with submodule conflicts, it's likely time to invest
in tooling to reduce the frequency of conflicts, change processes, or build
small tooling that helps resolve the problem.

## Conclusion
As I mentioned, Git has a learning curve and that is true with submodules. If
your build system or dependency system supports external dependencies, I would
go down that route. If those are not options, Git submodules can be a good
way to solve the problem - but become familiar with submodules, how they work
under the hood, and how to effectively resolve conflicts - define best practices
and write playbooks on what to do in what cases. Your teammates and stakeholders
will also need to be familiar with it, so: speak up and share your plans early.

I also want to thank the entire Chromium community for helping surface user
experience issues, and for opportunity to improve it.

_Note: Did I miss something? Feel free to reach out to me and I'll do my best to
update this post_.
