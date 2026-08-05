# Use an existing SSH key + manage multiple keys
<!-- verified: 2026-08 -->

## Load a key into the agent for this terminal session
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_<PURPOSE>
```
Example:
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_personal
```

**Why it bites:** `ssh-agent -s` on its own starts a *new* agent process and prints env vars — running it directly (without `eval`) in every new terminal tab leaves those vars unset in that shell, so `ssh-add` in one tab loads a key that a different tab's shell never sees. Always wrap it in `eval "$(ssh-agent -s)"`.

## Running multiple identities (e.g. personal + work) without re-adding every time
Add to `~/.ssh/config`:
```
Host github.com-<PURPOSE>
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_<PURPOSE>
    IdentitiesOnly yes
```
Example:
```
Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```
Clone using the alias host, not the plain `github.com`:
```
git clone git@github.com-<PURPOSE>:<ORG>/<REPO>.git
```

**Why it bites:** without `IdentitiesOnly yes`, ssh tries every key already loaded in the agent against the server, in order, and GitHub authenticates you as whichever key matches first — not necessarily the one you meant. You can end up pushing as the wrong GitHub account with no error at all, just a confusing author on the commit afterward.

## Check which key was actually offered
```
ssh -vT git@github.com-<PURPOSE> 2>&1 | grep "Offering public key"
```
