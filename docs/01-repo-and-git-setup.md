# 01: Repo and git setup

## Goal

Create this repo, and be able to `git pull` and `git push` from the server, where all the course work happens.

## What we did

### 1. Created the repo (from the laptop)

```bash
echo "# cloud-infra-learning" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/ChinmayNoob/cloud-infra-learning.git
git push -u origin main
```

### 2. Cloned it on the server

The repo was cloned to `~/cloud-infra-learning` on the server over HTTPS. Pulling worked, because the repo is public, but pushing failed:

```
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

The server already had an SSH key (`~/.ssh/id_ed25519`), but it belongs to a **different GitHub account**, so it can't push to a ChinmayNoob repo.

### 3. Added a dedicated SSH key for the ChinmayNoob account

On the server:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_chinmaynoob -N "" -C "coz-server (ChinmayNoob)"
cat ~/.ssh/id_ed25519_chinmaynoob.pub   # copy this
```

We added the public key to GitHub under **ChinmayNoob → Settings → SSH and GPG keys → New SSH key**, with the title `coz-server`.

### 4. Added a host alias so this repo uses that key

`~/.ssh/config` on the server:

```
# ChinmayNoob GitHub account (used by ~/cloud-infra-learning)
Host github-chinmaynoob
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_chinmaynoob
    IdentitiesOnly yes
```

`IdentitiesOnly yes` stops SSH from offering the other account's key first.

### 5. Pointed the repo at the alias and set the commit identity

```bash
cd ~/cloud-infra-learning
git remote set-url origin git@github-chinmaynoob:ChinmayNoob/cloud-infra-learning.git
git config user.name  "Chinmay Sawant"            # repo-local, not --global
git config user.email "chinmaysawant@cozclub.com"
```

We set the identity for this repo only, so other repos on the server keep their own identity.

## Result

```
$ ssh -T git@github-chinmaynoob
Hi ChinmayNoob! You've successfully authenticated, but GitHub does not provide shell access.

$ git pull
Already up to date.
```

Pull and push both work from the server.

## Tips

- To clone any other ChinmayNoob repo on the server, use the alias: `git clone git@github-chinmaynoob:ChinmayNoob/<repo>.git`
- The laptop copy still uses HTTPS. Run `git pull` there before editing, so it doesn't fall behind.

## Next

[02: Prerequisites](02-prerequisites.md)
