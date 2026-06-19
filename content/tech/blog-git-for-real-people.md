---
title: "Git for Real People: From Your First Commit to Surviving Merge Conflicts"
description: "A messy, honest guide to using Git in a community project without wanting to throw your laptop"
slug: "git-for-real-people"
date: "2026-06-19"
tags: ["git", "github", "open-source", "beginners", "merged"]
---

I've been using Git for about four years now and I still occasionally do things that make me go "what the hell was I thinking?"

This is not *that kind of guide*. You know the ones — they exist by the thousands and most of them are written by people who assume you've never used a terminal before or — more annoyingly — people who assume you already know everything and just need a cheatsheet. This is somewhere in the middle.

I watched [Mosh Hamedani's Git tutorial](https://www.youtube.com/watch?v=8JJ101D3knE) a while back and it's pretty good for the basics. But here's the thing — knowing the commands and actually surviving a real project with other humans are two very different skills. Kinda like the difference between learning to swim in a pool and being dropped in the middle of the ocean.

Anyway. Here's what I've learned the hard way.

## The Absolute Floor

If you only know five things, know these:

- `git init` — makes a folder a repo. You only do this once per project.
- `git add` and `git commit` — staging and saving your work locally. I lump these together because you almost never do one without the other.
- `git push` and `git pull` — sending and receiving changes from GitHub or wherever.
- `git branch` and `git checkout` — working on separate lines of development.
- `git merge` — joining those lines back together.

Everything else is gravy. You can get shockingly far with just these.

But here's a mistake I see beginners make all the time — they commit everything to `main` or `master` directly. Don't. I did this for my first six months and every time I needed to try something experimental, I had to either delete code or stash it somewhere. Branches exist precisely so you can break things without consequences.

I got sidetracked. Back to basics.

## Forking vs Cloning — There's a Difference

When you find a project on GitHub you want to contribute to:

- **Fork** = your personal copy of someone else's repo on GitHub. You own it. You can mess it up.
- **Clone** = downloading a repo to your computer.

The flow is: fork → clone your fork → make changes → push to your fork → open a Pull Request.

Where people trip: they fork, clone, make changes, then try to push directly to the original repo. Git will yell at you because you don't have write access.

Fork first, kids.

## Things I've Done So You Don't Have To

### Committing straight to main

I already mentioned this but it deserves its own bit because I've done it like twenty times. You're on main, you make a small change, commit, push. Next thing you know you're working on something half-baked and someone else pulls your incomplete code. Now their app is broken too. Congratulations, you're the reason the team meeting happened.

**Fix:** `git checkout -b fix/login-button` before touching anything. Always. Train your fingers.

### The dread merge conflict

This happens when two people change the same part of the same file. Git doesn't know whose version to keep. So it puts both in the file and makes you decide.

The first time I saw one I literally deleted the whole file and re-cloned. Don't do that.

**What actually happens:** Git marks the conflict with `<<<<<<<`, `=======`, and `>>>>>>>` markers. You open the file, find those markers, decide which code to keep (or blend both), delete the markers, save, `git add`, `git commit`. Done.

The video shows this pretty clearly. Open the file, look for the mess, clean it up, move on. It's not scary after the second time.

### Messing up a rebase

Rebasing is basically "I want to pretend I've been working on top of the latest code all along." It rewrites history. If you've already pushed your branch and someone else is working on it, rebasing will ruin their day.

**Golden rule:** rebase your own local branches only. Never rebase shared branches. If you're not sure, just merge instead of rebase. Merge is safer even if the history is a little uglier.

I don't think I'd go as far as the people who say "never rebase" — that's dramatic. But know what you're getting into.

### Push before pulling

You made changes, someone else pushed theirs first, now Git says *"rejected — fetch first."* This is normal. Pull their changes first, resolve any conflicts locally, then push.

Unless you're doing this:

```
git add .
git commit -m "wip"
git pull
git push
```

That works 90% of the time. The 10% is when conflicts happen and you need to do the <<<<<<< dance I mentioned above.

### `.gitignore` neglect

You don't want to commit environment variables, API keys, compiled binaries, or that `node_modules` folder. A good `.gitignore` at the start of the project saves headaches later.

**Speaking from recent personal experience:** we had a security incident in one of my repos because someone (me, it was me) committed a `.env` file with actual API keys in it. We had to rewrite git history to purge it. If the repo was public, anyone could've seen those keys by checking git log. Don't be me. Add `.env` to `.gitignore` on day one. Check the file in.

## Working With a Team

Git is the tool. The hard part is the people.

**Pull Requests are conversations.** When you open a PR, you're not just submitting code — you're starting a discussion. People will leave comments. Some will be useful. Some will feel nitpicky. Breathe.

**Review before merging.** I review my own PRs now before I even assign reviewers. You'd be surprised how many bugs you catch when you read your code fresh.

**Write decent commit messages.** "fixed stuff" is not a message. Neither is "Update file.txt." Spend 10 seconds writing something meaningful. Your future self will thank you when you're reading git blame at 2 AM.

## When Things Go Properly Wrong

Sometimes you `git merge` and everything looks fine until someone runs the tests. Or you force-pushed the wrong branch. Or you realized that the feature branch you've been working on for three days was branched off an outdated main.

**`git reflog`** — this is your emergency exit. Git keeps a log of every movement of HEAD. If you lose something, `git reflog` shows where HEAD was. Use `git reset --hard HEAD@{n}` to go back. But only do this on your local repo. Same rule as rebase — don't rewrite history other people are relying on.

## Blame Games and the Art of Finding Who Broke It

This might be the most underrated Git skill nobody talks about in beginner tutorials. You'll pick up `add`, `commit`, and `push` in a week. Reading logs and tracing bugs back to their origin? That takes years of being woken up at 3 AM for a production issue.

Here's the scenario: you're on-call, someone reports a bug in production, and you need to find out which change broke it and revert it fast. You don't have time to read every PR. You need the smoking gun.

### `git log` — but with taste

Plain `git log` is fine for casual browsing. For debugging, you want these:

```
git log --oneline --graph --decorate --all
```

Shows the whole branching picture – who branched from where, which merges happened, where two lines of work converged. I alias this to `git lol` because it looks like a graph of laughing commits.

But the real money is:

```
git log -p --follow -S "functionName" -- filename.js
```

`-S` searches the diff for a specific string appearing or disappearing. If a variable changed from `isAdmin` to `hasAdminRole` and suddenly auth is broken, `-S "isAdmin"` shows you every commit that touched that string. No more scrolling through irrelevant commits.

### `git blame` is not an accusation, I promise

I know the name sounds aggressive. `git blame` shows you who last changed each line of a file, and in which commit. It's not about pointing fingers — it's about finding context.

```
git blame app/controllers/auth.rb
```

You see commit hashes and authors next to every line. Grab the hash, run `git show <hash>` to see the full diff and — if the team writes decent messages — the *why* behind the change.

The actual flow for debugging:

1. Find the error in your logs or monitoring
2. Identify the file and line where the error originates
3. `git blame` that file, find the commit that introduced the bad line
4. `git log -1 <hash>` to read the full commit message and PR reference
5. Now you have context: was it a rushed hotfix? A misunderstood requirement? A merge that incorrectly resolved a conflict?
6. Decide: revert the commit (`git revert <hash>`) or patch forward

And yeah, sometimes `git blame` points back to you. That's fine. It's better to know fast and fix fast than spend four hours debugging while pretending you didn't write that line.

Also — I once spent an hour blaming the wrong person because I forgot I was on a different branch. True story.

### Common gotcha: whitespace

If someone ran an auto-formatter across the file, `git blame` will point at the formatter commit for every line, even though the actual logic change happened months earlier. Use `git blame -w` to ignore whitespace changes. It skips over the formatting commit and shows the real author.

### `git bisect` — magic

You have a working commit (good) and a broken commit (bad). `git bisect` does a binary search through the history.

```
git bisect start
git bisect bad HEAD           
git bisect good v2.0.0
```

Git checks out a commit halfway between. You test it, mark it good or bad, and it narrows down until it finds the exact commit that introduced the bug. For a repo with 1000 commits between good and bad, that's about 10 checkouts instead of 1000.

I automate it with `git bisect run` when I can write a test script that returns 0 for good and non-zero for bad. Let the computer do the boring work.

None of these skills are flashy. But when the production pager goes off and everyone's looking at you, knowing how to trace a line of code back to the PR that introduced it is worth more than all the merge conflict tricks in the world.

## One More Thing About Branches

I forgot to mention this earlier and it's bugging me so here it is — naming branches. Please name them something meaningful. Not `branch2`, not `new-branch`, not `test`. Call it `fix/payment-timeout` or `feat/user-dashboard`. It makes a massive difference when you're looking at a list of twenty branches and trying to remember which one does what.

I've been guilty of `fix-fix-real-one-this-time`. We all have our moments.

## The Part Where I Admit I'm Still Learning

I've been using Git for years and I still google "how to undo a git commit" at least once a month. The command is `git reset HEAD~1` by the way. I should know that by now. I don't. I look it up every single time.

The whole point of Git is that you can mess up and recover. Branch off, commit early and often, push when you're ready, and when everything falls apart — `git reflog` has your back.

Or just delete your local repo and clone fresh. Works too. I've done that more times than I'll admit.
