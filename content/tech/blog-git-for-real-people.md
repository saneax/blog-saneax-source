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

1. `git init` — makes a folder a repo. You only do this once per project.
2. `git add` and `git commit` — staging and saving your work locally.
3. `git push` and `git pull` — sending and receiving changes from GitHub or wherever.
4. `git branch` and `git checkout` — working on separate lines of development.
5. `git merge` — joining those lines back together.

Everything else is gravy. You can get shockingly far with just these.

But here's a mistake I see beginners make all the time — they commit everything to `main` or `master` directly. Don't. I did this for my first six months and every time I needed to try something experimental, I had to either delete code or stash it somewhere. Branches exist precisely so you can break things without consequences.

I got sidetracked. Back to basics.

## Forking vs Cloning — There's a Difference

When you find a project on GitHub you want to contribute to:

- **Fork** = your personal copy of someone else's repo on GitHub. You own it. You can mess it up.
- **Clone** = downloading a repo to your computer.

The flow is: fork → clone your fork → make changes → push to your fork → open a Pull Request.

Where people trip: they fork, clone, make changes, then try to push directly to the original repo. Git will yell at you because you don't have write access. Fork first, kids.

## Common Mistakes I've Made So You Don't Have To

### 1. Committing straight to main

I already mentioned this but it deserves its own section because I've done it like twenty times. You're on main, you make a small change, commit, push. Next thing you know you're working on something half-baked and someone else pulls your incomplete code. Now their app is broken too. Congratulations, you're the reason the team meeting happened.

**Fix:** `git checkout -b fix/login-button` before touching anything. Always. Train your fingers.

### 2. The dreaded merge conflict and immediate panic

Merge conflicts happen when two people change the same part of the same file. Git doesn't know whose version to keep. So it puts both in the file and makes you decide.

The first time I saw one I literally deleted the whole file and re-cloned. Don't do that.

**What actually happens:** Git marks the conflict with `<<<<<<<`, `=======`, and `>>>>>>>` markers. You open the file, find those markers, decide which code to keep (or blend both), delete the markers, save, `git add`, `git commit`. Done.

The video shows this pretty clearly. Open the file, look for the mess, clean it up, move on. It's not scary after the second time.

### 3. Messing up a rebase

Rebasing is basically "I want to pretend I've been working on top of the latest code all along." It rewrites history. If you've already pushed your branch and someone else is working on it, rebasing will ruin their day.

**Golden rule:** rebase your own local branches only. Never rebase shared branches. If you're not sure, just merge instead of rebase. Merging is safer even if the history is a little uglier.

I don't think I'd go as far as the people who say "never rebase" — that's dramatic. But know what you're getting into.

### 4. Push before pulling and getting rejected

You made changes, someone else pushed theirs first, now Git says *"rejected — fetch first."* This is normal. Pull their changes first, resolve any conflicts locally, then push.

Unless you're doing this:

```
git add .
git commit -m "wip"
git pull
git push
```

That works 90% of the time. The 10% is when conflicts happen and you need to do the `<<<<<<<` dance I mentioned above.

### 5. `.gitignore` neglect

You don't want to commit environment variables, API keys, compiled binaries, or that `node_modules` folder. A good `.gitignore` at the start of the project saves headaches later.

**Speaking from recent personal experience:** we had a security incident in one of my repos because someone (me, it was me) committed a `.env` file with actual API keys in it. We had to rewrite git history to purge it. If the repo was public, anyone could've seen those keys by checking git log. Don't be me. Add `.env` to `.gitignore` on day one. Check the file in.

## Working With a Team — The Social Layer

Git is the tool. The hard part is the people.

**Pull Requests are conversations.** When you open a PR, you're not just submitting code — you're starting a discussion. People will leave comments. Some will be useful. Some will feel nitpicky. Breathe.

**Review before merging.** I review my own PRs now before I even assign reviewers. You'd be surprised how many bugs you catch when you read your code fresh.

**Write decent commit messages.** "fixed stuff" is not a message. Neither is "Update file.txt." Spend 10 seconds writing something meaningful. Your future self will thank you when you're reading git blame at 2 AM.

## When Things Go Properly Wrong

Sometimes you `git merge` and everything looks fine until someone runs the tests. Or you force-pushed the wrong branch. Or you realized that the feature branch you've been working on for three days was branched off an outdated main.

**`git reflog`** — this is your emergency exit. Git keeps a log of every movement of HEAD. If you lose something, `git reflog` shows where HEAD was. Use `git reset --hard HEAD@{n}` to go back. But only do this on your local repo. Same rule as rebase — don't rewrite history people are relying on.

## The Part Where I Admit I'm Still Learning

I've been using Git for years and I still google "how to undo a git commit" at least once a month. The command is `git reset HEAD~1` by the way. I should know that by now. I don't. I look it up every single time.

The whole point of Git is that you can mess up and recover. Branch off, commit early and often, push when you're ready, and when everything falls apart — `git reflog` has your back.

Or just delete your local repo and clone fresh. Works too. I've done that more times than I'll admit.
