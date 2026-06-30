---
title: "Ansible Like a Pro — Part 1: Why Ansible Won Config Management (and What Came Before)"
description: "An opinionated look at configuration management tools — Puppet, Chef, Salt, CFEngine — and why Ansible's agentless SSH model + execution environments + ansible-navigator is the modern right answer."
slug: "ansible-pro-part1"
date: "2026-06-20"
tags: ["ansible", "ansible-navigator", "execution-environments", "containers", "config-management", "devops", "tutorial-series"]
---

**Series: Ansible Like a Pro**

- **Part 1** ← You are here — Config management landscape, why Ansible won, architecture of ansible-navigator and execution environments
- **[Part 2: Building Your First Execution Environment](https://blog.saneax.in/tech/ansible-pro-part2/)** — *Coming next* — Containerfile, ansible-builder, Ubuntu-based EE from scratch
- **[Part 3: Writing Your First Playbook with Navigator](https://blog.saneax.in/tech/ansible-pro-part3/)** — *Planned*

---

I've been doing config management for over a decade now. Puppet in 2012. Chef in 2014. Salt around 2016. Ansible since 2017. I've got the scars.

Here's the thing nobody tells you about the config management wars: **they're over. Ansible won.**

Not because it's technically the best at everything — it's not. Salt is faster at pushing. Puppet has better state enforcement. Chef has better community cookbooks (or had, before the exodus).

But Ansible won because of one thing: **SSH is already there.**

No agent. No certificate authority to manage. No daemon to keep alive. No package to install on 5000 servers before you can do anything. Your SSH keys work or they don't, and if they don't, that's a problem you already know how to fix.

## The Config Management Graveyard (Or: What You're Not Missing)

### Puppet

Puppet was the first tool I used professionally. Ruby DSL. Master-agent architecture. You wrote manifests on the master, agents pulled them periodically. The pull model meant changes didn't happen until the agent checked in — every 30 minutes by default.

The good parts:
- Good resource abstraction model
- Facter was genuinely useful for inventory
- The dependency graph (`before`, `require`, `notify`, `subscribe`) was clever

The parts that made me quit:
- Certificate management was a nightmare. Expired certs meant agents stopped talking to the master. Auto-signing was a security hazard. Manual signing didn't scale.
- The Ruby DSL was fragile. One syntax error in a manifest and the whole environment broke.
- Master scaling: you needed PuppetDB, Puppetserver tuning, Passenger/nginx config, all before you could manage more than a thousand nodes.
- Upgrades were terrifying. Puppet 3 to 4 was basically a rewrite.

### Chef

Chef took a different approach — Ruby DSL with a more procedural style (resources in order, not a graph). Chef Server + Chef Client. Workstations with `knife`.

The good parts:
- Ohai (like Facter) was solid
- Test Kitchen + Berkshelf + ChefSpec was advanced for its time
- The community cookbook ecosystem was massive

The parts that made me quit:
- Same agent problem as Puppet — you had to install something on every node
- Chef Server was a Java monolith. Reliable, but heavy.
- The Ruby learning curve was steeper than Puppet's. You weren't writing config files, you were writing Ruby that generated config files.
- `knife` commands were cryptic. I never memorized them. I always had to Google "delete node chef"

### SaltStack

Salt is interesting because it tried to solve the speed problem. Instead of SSH, it uses ZeroMQ for a pub/sub message bus. Push-based, not pull-based. Fast.

The good parts:
- Blazing fast. Push to 10,000 nodes in seconds.
- `salt-call` standalone mode (agentless, like Ansible)
- State files in YAML-ish format (called SLS, kinda like Ansible playbooks)
- Event-driven automation with Reactor

The parts that made me quit:
- ZeroMQ was a dependency nightmare. Different versions on master and minion? Everything breaks. Silently.
- The YAML was almost YAML but not quite. Salt's custom renderers meant you couldn't just use standard YAML parsers.
- The config files (`master`, `minion`, `pillar`) had bespoke formats that you memorized or you couldn't troubleshoot anything.
- The community fractured after VMware acquired SaltStack and then Broadcom aquired VMware. Lots of people jumped ship.

### CFEngine

CFEngine is the granddaddy — first released in 1993. It still exists. It's still used in places where you absolutely cannot tolerate a dependency (it's written in C, almost no runtime dependencies). But writing CFEngine policy is like writing C with different syntax.

I tried it once. I gave up after three days and went back to shell scripts.

## Why Ansible Won

When Red Hat acquired Ansible in 2015, I was skeptical. "They'll ruin it," I thought. "They'll make it into a Puppet clone with a subscription model."

I was wrong. Ansible Inc. and later Red Hat made a few key bets that turned out to be right:

### 1. Agentless over agent-based

No daemon to install. No port to open (SSH is already open). No certificate lifecycle to manage. You install Ansible on ONE machine (your control node), write a playbook about what you want done, and it SSHes into everything else and does it.

This single decision is what pushed Puppet and Chef out of most new deployments by 2018. Why would you install an agent on 500 servers when you could just SSH in?

The trade-off: it's slower. SSH handshake per-connection adds latency. For one-off operations on 50 servers, you don't notice. For push-to-10k-servers scenarios, you notice. Ansible handles this with forks (`-f 50`), but it's still not Salt-fast.

And honestly? Most teams don't have 10k servers. Most teams have 50 to 500. For that range, Ansible's speed is fine.

### 2. YAML over DSL

Puppet had Ruby. Chef had Ruby + a custom DSL. Salt had custom YAML-ish.

Ansible said: plain YAML. You can run any valid YAML through Ansible. If you've ever written a Docker Compose file or a Kubernetes manifest, you already know Ansible syntax.

```yaml
- name: Install and configure nginx
  hosts: webservers
  become: yes
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Copy config file
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
```

Three lines of metadata. Four lines of task. This is readable by someone who's never used Ansible. That's the whole point.

### 3. `ansible-pull` when you need the agent model

Ansible's default push model works for most cases. But if you need pull-based (ephemeral containers, auto-scaling groups, disconnected environments), `ansible-pull` exists. It runs as a cron job, checks out your playbooks from git, and applies them locally.

Best of both worlds. Push by default, pull when you need it.

### 4. Modules over monolithic code

Ansible modules are standalone executables. In the old days, they were shell scripts that output JSON. Today they're Python scripts, but the contract is the same:

1. Run a command
2. Output JSON to stdout
3. Return an exit code

This means anyone can write a module. I've written modules in Bash, Python, and Go. You can even write them in whatever language you want, as long as they speak JSON on stdout.

The module ecosystem is enormous now. AWS, Azure, GCP, network gear, containers, databases, monitoring — almost everything has a first-party or community module.

### What Ansible is bad at (let's be honest)

- **State enforcement without continuous runs:** Ansible doesn't run continuously. You run a playbook, it converges, it exits. If someone SSHes in and messes with a config file two minutes later, Ansible won't fix it until you run the playbook again. Puppet agents ran every 30 minutes and fixed drift automatically. Ansible requires an external trigger (cron, CI/CD, AWX/AAP) for the same behavior.

- **Windows management:** It works. But it's not as good as Linux. WinRM is not SSH. The module coverage is thinner.

- **Large-scale performance:** At 1,000+ nodes with complex plays, the `gather_facts` phase alone can take minutes. You can mitigate with `gather_facts: no`, centralized fact caching, and tuning forks, but you shouldn't have to.

- **Stateful operations:** Ansible is inherently stateless — it runs tasks, checks results, exits. If a task fails halfway through a playbook, there's no automatic rollback. You write error handling yourself.

I still use Ansible. Every day. These are the limitations I've learned to work around, not dealbreakers.

## Enter the Modern Era: Execution Environments and ansible-navigator

If Ansible's old model had a problem, it was **dependency hell on the control node.**

Your playbook needs `community.docker` collection. But your coworker's playbook needs `community.docker` version 3.x while you need 2.x. You need Python 3.10 for one, 3.12 for another. And you're sharing the same `ansible` command installed via pip on a shared jump box.

I've seen this exact situation cause a production outage. Someone ran `pip install ansible` to get a new collection, which bumped `cryptography` from 41 to 43, which broke another team's playbook that pinned an older version. Chaos.

### Execution Environments (EEs)

The solution is containers. Ansible Execution Environments are container images that include:

- `ansible-core` (the runtime)
- `ansible-runner` (the interface layer)
- Python with specific version
- Collections you need
- System packages you explicitly add (openssh-clients, sshpass, etc.) — these don't ship in the base image
- Python dependencies

Everything in one image. Tagged. Immutable. Versioned.

So instead of:

```bash
# Old way - hope your Python env is right
pip install ansible ansible-navigator
ansible-playbook -i inventory playbook.yml
```

You do:

```bash
# New way - containerized, reproducible
ansible-navigator run playbook.yml --pull-policy missing
```

Navigator pulls the EE image once (cached locally), runs your playbook inside it, and cleans up. Your host system stays clean. Your coworker's Python dependencies don't conflict with yours.

### The Architecture

Here's how the pieces connect:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Your Laptop / CI Runner                      │
│                                                                      │
│  ┌──────────────────────┐                                            │
│  │   ansible-navigator   │  CLI + TUI for running Ansible content   │
│  │  (CLI / TUI / stdout) │                                            │
│  └──────────┬───────────┘                                            │
│             │                                                        │
│             │ invokes                                                 │
│             ▼                                                        │
│  ┌──────────────────────┐                                            │
│  │    ansible-runner     │  Handles stdout, artifacts, status codes   │
│  │  (Python library)     │  Abstractions over ansible-core           │
│  └──────────┬───────────┘                                            │
│             │                                                        │
│             │ runs inside                                            │
│             ▼                                                        │
│  ┌─────────────────────────────────────────┐                         │
│  │       Execution Environment (EE)         │                         │
│  │  ┌─────────┐ ┌──────────┐ ┌──────────┐ │                         │
│  │  │ansible- │ │Python     │ │Collections│ │                         │
│  │  │core     │ │3.x + deps │ │+ sys pkgs │ │                         │
│  │  └─────────┘ └──────────┘ └──────────┘ │                         │
│  │         Container (podman/docker)       │                         │
│  └──────────────────┬──────────────────────┘                         │
│                     │                                                 │
│                     │ SSH (via sshpass / keys)                       │
└─────────────────────┼───────────────────────────────────────────────┘
                      │
                      ▼
         ┌─────────────────────────────────┐
         │       Target Servers             │
         │  (bare metal, VMs, containers)   │
         │  No agent needed                 │
         │  SSH + Python is all you need    │
         └─────────────────────────────────┘
```

**The container engine** — podman (rootless, no daemon) or docker — runs the EE on your control node. The EE contains ansible-core AND ansible-runner. ansible-runner provides the interface between navigator and the actual playbook execution, handling return codes, artifact collection, and logging.

**ansible-navigator** itself runs on your host (installed via `pip` or `ansible-dev-tools`). It manages the EE lifecycle — pulls images, spawns containers, and provides the TUI interface.

## Setting Up: Fedora vs Ubuntu

I'm going to cover both because I know people use both. But my honest opinion: **Fedora is a smoother experience for Ansible work.** Better Python integration, podman default, newer kernel, ansible-dev-tools packages in the repos. Ubuntu works fine, but you'll do more manual setup.

### Fedora 41 (the easy way)

```bash
# Install ansible-dev-tools (a Python meta-package) with pipx
# (pipx ships in the Fedora repos — sudo dnf install pipx — if you don't have it)
pipx install ansible-dev-tools

# Verify
ansible-navigator --version

# Navigators first run pulls the demo EE automatically
ansible-navigator
```

That's it. `ansible-dev-tools` is a meta-package that pulls in ansible-core, ansible-navigator, ansible-lint, molecule, and the rest. There's no `ansible-dev-tools` RPM in Fedora — it's a Python package, so `pipx` is the recommended path.

### Ubuntu 24.04 (the manual way)

```bash
# Install pip if you don't have it
sudo apt update
sudo apt install python3-pip python3-venv podman

# Install ansible-navigator (in a venv, because dependencies)
python3 -m venv ~/ansible-env
source ~/ansible-env/bin/activate

# Install navigator
pip install ansible-navigator

# Verify
ansible-navigator --version

# Optional: create a symlink for convenience
sudo ln -sf ~/ansible-env/bin/ansible-navigator /usr/local/bin/
```

Why `podman` over `docker` on Ubuntu? Rootless containers by default, no daemon, no `docker` group, no `sudo` needed. Works better in CI environments. ansible-navigator uses podman by default if it's available.

**Make sure you have SSH keys set up:**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_ed25519
ssh-copy-id -i ~/.ssh/ansible_ed25519 user@target-server
```

Navigator uses your host's SSH config and keys. It bind-mounts them into the EE container automatically (and forwards your ssh-agent socket) — you don't need to bake keys into the image.

## Building a Minimal Ubuntu-Based EE

Let's build an EE we'll use for the rest of this series. An Ubuntu base, minimal, with the collections we need.

Create a project directory and the EE definition:

```bash
mkdir -p ~/ansible-project/ee
cd ~/ansible-project/ee
```

Create `execution-environment.yml`:

```yaml
---
version: 3

images:
  base_image:
    name: docker.io/ubuntu:24.04

dependencies:
  python_interpreter:
    package_system: python3
  ansible_core:
    package_pip: ansible-core>=2.17
  ansible_runner:
    package_pip: ansible-runner
  system:
    - openssh-client
    - sshpass
    - python3-pip
    - python3-venv
    - ca-certificates
    - curl
  galaxy:
    collections:
      - ansible.utils
      - ansible.posix
      - community.general
  python:
    - PyYAML
    - jmespath

options:
  package_manager_path: /usr/bin/apt-get
```

Build it:

```bash
# Install ansible-builder (only needed on the build machine)
pip install ansible-builder

# Build the EE (use --container-runtime docker if you're on docker)
ansible-builder build --tag ansible-ubuntu-ee:latest

# Verify
podman images | grep ansible-ubuntu-ee
```

The first build takes a few minutes (downloading Ubuntu base + installing packages). Subsequent builds are faster because layers are cached.

**Important:** `ansible-builder` is separate from `ansible-navigator`. You build EEs with builder (once), then run them with navigator (every day). You don't need builder on every machine — just on the one where you maintain images.

Run a playbook with your new EE:

```bash
ansible-navigator run playbook.yml \
  --eei localhost/ansible-ubuntu-ee:latest \
  --pull-policy never \
  --mode stdout
```

- `--eei`: which execution environment image to use
- `--pull-policy never`: don't try to pull from a registry (we built it locally)
- `--mode stdout`: don't open the TUI, just run and print

## What We've Built

By now you should have:

1. A strong opinion on why Ansible outlasted Puppet, Chef, and Salt
2. An understanding of execution environments — what problem they solve, how they work
3. ansible-navigator installed on Fedora or Ubuntu
4. A custom Ubuntu-based EE built and tested

**Next in the series:** [Part 2: Building Your First Execution Environment](https://blog.saneax.in/tech/ansible-pro-part2/) — We'll go through the whole `execution-environment.yml` file in detail, add custom collections, handle Python dependencies, set up private repos, and set up a local container registry so your team can share EEs without a registry server.

---

*This series assumes you're a seasoned sysadmin. If you don't know what SSH keys are, or what a container is, go read those primers first and come back.*
