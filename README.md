# Git Demos

This repo contains branches used in the demos of the presentation 'Git & U.' All the commands used for each demo will be reproduced in this README.

This repo is technically a fork of the [python adventure](https://github.com/brandon-rhodes/python-adventure) project. The changes made in each "dev" branch are miniscule and somewhat pointless, but they make for a more entertaining demo amongst ~~nerds~~ programmers.


## 'Undoing' Merges


### Setting Up the Bad Revert Demo

```bash
git switch master
git switch -c rel/v1.1-rc
git merge dev/w001-change-magic --no-ff
git merge dev/w002-start-in-dark
```
(We're using the `--no-ff` flag to force a merge commit. It's not necessary for the second merge because the `dev/w002-start-in-dark` branch and the `rel/v1.1-rc` branch can't be merged as a "fast-forward" merge after we added a commit to `rel/v1.1-rc` that `dev/w002-start-in-dark` doesn't have.)

In this scenario, let's say the repo maintainer has to "undo" the merge of `dev/w001-change-magic`. This could be for several reasons, but let's just say that the repo maintainer doesn't want those changes to be released until _next_ week. Because it's the entire point of this demo, let's assume that the repo maintainer going to revert the merge of `dev/w001-change-magic` into `rel/v1.1-rc` when preparing the release branch, like so:

```bash
git revert db1bb1e03f7f55fde66b58a4de01567856a94e72 -m 1
```
`db1bb1e03f7f55fde66b58a4de01567856a94e72` is the commit hash of _my_ local merge commit while writing this up. If you're running through all these steps yourself, you'll have a different commit hash you need to revert. The easiest way to find it is to use `git log` and look for the commit with the message of "Merge branch 'dev/w001-change-magic' into rel/v1.1-rc". If you followed the steps exactly, then you can get away with using `HEAD~1`.

> **NOTE:** Even though this isn't what you should do in this exact situation, when you are reverting a merge, you should probably revert the _merge commit_ instead of the changes in the topic branch. This is because there could be multiple commits in the branch; if there were multiple commits and you weren't reverting the merge commit, you'd have to individually revert each commit specific to that branch. Ain't nobody go time for that!

Assuming that the release went well, the repo maintainer would run something like the following commands:

> **NOTE:** If you're following along and haven't setup `gpg`/`gpg2`, then just do `git tag v1.1` instead.
```bash
# assuming on rel/v1.1-rc 
git tag -s v1.1 -m "Signed v1.1 tag, because I'm cool"
git push origin v1.1
git switch master
git merge rel/v1.1-rc
git push
git switch develop
git merge master
git push
```

Given the Selective Git Flow process (where the release is merged into `master` and `master` is merged into `develop`), this will merge that revert commit into `develop`, too!

This could cause a headache for the developer responsible for the `dev/w001-change-magic` branch, as testing would fail since the changes would no longer be present. And, if they didn't know about the revert commit, it could really ... not be swell. They might try to continually merge the dev branch into `develop`, but get the "Already up to date" message and be confused.


### Fixing It After it Happened

Let's say you're the developer of `dev/w001-change-magic`, and you've been told by a bot or a QA department that your changes have failed testing in `develop`.

Given that the repo maintainer already messed things up, the easiest way to fix everything at this point is to do the following:

```bash
git fetch
git checkout master
git pull
git checkout dev/w001-change-magic
git merge master
```
At this point, you might have noticed that the changes are now missing in the `dev/w001-change-magic` branch. This is because we merged the `master` branch, which was updated by the maintainer to include that nasty revert commit.

What ever will we do?

Revert the revert!

Use `git log` to find the hash of the revert commit that made your changes completely disappear. Let's say it's `4ec80f75f8ea1c36fd23f7424698034d00943ef7`.

All you do is the following:
```bash
git revert 4ec80f75f8ea1c36fd23f7424698034d00943ef7
git push
```
And voilà! Now you can either create a PR or manually merge the updated `dev/w001-change-magic` branch into `develop` and the changes will return!

### What Should the Repo Maintainer Have Done Instead?

Instead of using `git revert` to create another commit that undoes the changes introduced by the `dev/w001-change-magic` branch, the repo maintainer "should" have taken a different approach.
 
I say "should" in quotation marks because this depends on a few things (how many merges occurred after `dev/w001-change-magic` that also had merge conflicts (and whether or not the repo maintainer is has `rerere` enabled).

In the best-case situation &mdash; such as this demo ;) &mdash; the repo maintainer would simply have to do a **rebase** and drop the merge of the branch that shouldn't be in the release.

Assuming on `rel/v1.1-rc` (if not, switch back to it), use `git log` to figure out how many commits back it was. In our case, it was 1 before our current position.

The command we will run is:
```bash
git rebase -ri HEAD~1^
```

This will open something very similar to the following in the default text editor on your system:
```bash
label onto

# Branch dev/w001-change-magic
reset onto
pick cf03aed Change magic word
label dev/w001-change-magic

# Branch dev/w002-start-in-dark
reset onto
pick 378e620 Start in darkness
label dev/w002-start-in-dark

reset onto
merge -C db1bb1e dev/w001-change-magic # Merge branch 'dev/w001-change-magic' into rel/v1.1-rc
merge -C 4ec80f7 dev/w002-start-in-dark # Merge branch 'dev/w002-start-in-dark' into rel/v1.1-rc

# Rebase 2b9c3a6..4ec80f7 onto 4ec80f7 (10 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

Neverminding that this actually has instructions in the comments, what you want to do is navigate to this line:
```
merge -C db1bb1e dev/w001-change-magic # Merge branch 'dev/w001-change-magic' into dev/v1.1-rc
```

And simply delete it. :)
Save and close the file and Git should take care of the rest.

> **NOTE:** Rebasing rewrites history, so if you're already merged to `master`, pushed, and people started working off it it, it's generally too late (unless your team is small enough where everyone can easily delete their local copies of `master` and get the correct one.) 
> 
> If you're working in a branch that you haven't pushed yet, you should be fine to take this approach. If you have pushed the branch but no one important ;) is working on that same branch, you should be fine to just force push it.


### Fixing it When Things _Really_ Went Wrong

Sometimes the history of a branch gets really, really messed up. In this scenario, let's say that the repo maintainer did a bad revert when prepping a release, merged it into `master` and `develop`, and the developer responsible for `dev/w001-change-magic` was alerted, didn't know how to simply revert the revert and made a bunch of changes.

If the history of a branch is really messed up, you aren't without recourse. One of my favorite things to do in the worst-of-the-worst situations is identify a point in the branches history where the changes were correct (i.e., use `git log`).

Once you find the commit where things were okay (let's say it was `56046233892f9f96c9b2d9e4c849fe74c68906e6`), you'll do the following:
```bash
git switch -c tmp/good-times
# any time you meet a payment . . .
# any time you need a friend . . .
git checkout 56046233892f9f96c9b2d9e4c849fe74c68906e6 -- .
```
This last command (`git checkout 56046233892f9f96c9b2d9e4c849fe74c68906e6 -- .`) assumes you're in the project's root directory. It'll checkout all files from `56046233892f9f96c9b2d9e4c849fe74c68906e6`. This is nice because it's bringing over the raw files as they were, but with the current history.

In practice, you'd need to check _all_ the differences and make sure you're only getting what was messed up history-wise (due to bad use of `git revert` or bad merge conflict resolutions).


## 'Redoing' Merges

Let's assume that none of the stuff from the other demo happened, but instead bad merge conflict resolutions were made when `dev/w001-change-magic` was merged with `dev/w003-obfuscate-magic-word` in branch `develop`. Ideally, you want to catch this before you push `develop` and instead reset the bad merge, but if this happens and you can't rewrite history, do the following:
```bash
git switch dev/w003-obfuscate-magic-word
git pull # get latest changes
git switch dev/w001-change-magic
git pull # get latest changes
git switch -c mrg/w001/w003
git merge dev/w003-obfuscate-magic-word
# resolve conflicts
git commit
git switch develop
git merge mrg/w001/w003
```
Oh, no! We got the same conflicts again when merging `mrg/w001/w003` into `develop`! :O
```bash
git checkout --theirs adventure/advent.dat
git add adventure/advent.dat
# or, if you're feeling bold:
# git checkout --theirs .
# git add .
git commit
```

Be more careful next time! Check your changes before committing and pushing!


### Fixing Issues Before Committing

So, you're taking my advice and checking merges before committing and pushing? Hopefully you haven't had to learn this lesson the hard way. ;)

If you're checking the resulting code before committing and you've found a problem, you can remerge via whatever mergetool(s) you use for individual files like so:

```bash
git checkout -m -- <file-path>
```

To completely restart the merge, it's much easier:
```bash
git merge --abort
# <insert the git merge command you ran before here>
```


### Fixing Issues Before Pushing

So, you committed and after testing changes found an issue and need to do the merge over again?
```
git reset HEAD~1 --hard
```
And remerge!

> **NOTE:** Sad that you have to re-resolve all conflicts? Enable `rerere`, train using `rerere-train.sh` with your "bad merge" branch, and then re-merge. Then refer to the above section to remerge an individual file.
>
> (Went through the steps without reading ahead, reset `HEAD`, and now you don't know what the hash for the bad branch was? Use `git reflog`!)

---


## Original Python Adventure README

Welcome to the git repository for the Python 3 version of Adventure!
The project README is one level deeper, inside of the package itself:

[adventure/README.txt](adventure/README.txt)
