
# 1) On your Ubuntu server (once)

```bash
# Install git
sudo apt update && sudo apt install -y git

# Create restricted git account
sudo adduser --system --home /srv/git --group --shell /usr/bin/git-shell git
sudo -u git mkdir -p /srv/git/repos /srv/git/.ssh
sudo chmod 700 /srv/git/.ssh

# Add your SSH public key to the git user (from YOUR laptop)
# (replace user@SERVER with your sudo-capable account on the server)
cat ~/.ssh/id_ed25519.pub | ssh user@SERVER \
  "sudo tee -a /srv/git/.ssh/authorized_keys >/dev/null && \
   sudo chown git:git /srv/git/.ssh/authorized_keys && \
   sudo chmod 600 /srv/git/.ssh/authorized_keys"

# (Optional, if firewall)
sudo ufw allow OpenSSH
```

# 2) Create a bare repo on the server (one per project)

```bash
ssh user@SERVER "sudo -u git git init --bare --shared=group /srv/git/repos/MYREPO.git && \
                 sudo chgrp -R git /srv/git/repos && sudo chmod -R g+ws /srv/git/repos"
```

# 3) From your local repo: push everything to your server

```bash
cd /path/to/your/local/repo

# Add your server as a remote and mirror all refs (branches + tags)
git remote add backup git@SERVER:/srv/git/repos/MYREPO.git
git push --mirror backup
```

# 4) (Optional) Make your server the new origin & stop using GitHub

```bash
git remote set-url origin git@SERVER:/srv/git/repos/MYREPO.git
git remote remove backup   # or keep it
```

# 5) Test

```bash
git ls-remote origin   # or: git ls-remote backup
```

That’s it. You now have your own SSH Git host at `/srv/git/repos/*.git`, with a locked-down `git` user (`git-shell`) and all history mirrored off GitHub.


Here’s a quick, clean **smoke test** you can run end-to-end.

```bash
# ---- config ----
SERVER=your.server.or.ip
REPO=MYREPO
REMOTE="git@$SERVER:/srv/git/repos/$REPO.git"

# 0) SSH auth OK? (git-shell will deny interactive shell; that’s fine)
ssh -T git@$SERVER || true
# If you see something like “interactive shell is disabled” but no auth errors, you’re good.

# 1) Can Git see the remote refs?
git ls-remote "$REMOTE"

# 2) Fresh clone test
rm -rf /tmp/${REPO}-test
git clone "$REMOTE" /tmp/${REPO}-test
cd /tmp/${REPO}-test
git remote -v

# 3) Push/roundtrip test (commit -> push -> verify from a second clone)
BRANCH="$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo main)"
echo "$(date) roundtrip ok" >> _roundtrip.txt
git add _roundtrip.txt
git commit -m "roundtrip: $(date -u +%F\ %T) UTC"
git push origin "$BRANCH"

# 4) Verify from a separate fresh clone
cd /tmp
rm -rf ${REPO}-verify
git clone "$REMOTE" ${REPO}-verify
grep -n "roundtrip ok" ${REPO}-verify/_roundtrip.txt

# 5) Tags test (optional)
cd /tmp/${REPO}-test
git tag smoke-tag-$(date +%s)
git push origin --tags
git ls-remote --tags "$REMOTE"

# 6) Server-side sanity check (optional)
# (run on the server)
#   sudo -u git git for-each-ref --format='%(refname)' /srv/git/repos/$REPO.git
```

**Pass criteria**

* Step 1 shows refs (or nothing if it’s a new repo), no auth errors.
* Step 2 clones successfully.
* Step 3 push succeeds.
* Step 4 the `_roundtrip.txt` exists with your line.
* Step 5 tags are visible via `ls-remote --tags`.
* `ssh -T git@SERVER` refusing a shell is expected (that means the restricted `git-shell` is working).

