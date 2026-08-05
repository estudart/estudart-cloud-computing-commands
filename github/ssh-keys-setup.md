# Create a new SSH key and register it with GitHub
<!-- verified: 2026-08 -->

## 1. Generate the key
```
ssh-keygen -t ed25519 -C "<YOUR_EMAIL>"
```
Example:
```
ssh-keygen -t ed25519 -C "alex@nike.com"
```
When prompted for a file path, give it a name that says what it's for instead of accepting the default blindly:
```
/Users/<YOUR_USERNAME>/.ssh/id_ed25519_<PURPOSE>
```
Example: `/Users/alex/.ssh/id_ed25519_personal`

**Why it bites:** if a default-named key already exists and you hit enter without renaming, `ssh-keygen` silently offers to overwrite it. Say no, or you lose the old key.

## 2. Copy the public key to your clipboard
```
pbcopy < ~/.ssh/id_ed25519_<PURPOSE>.pub
```

## 3. Add it in GitHub (UI, step by step)
1. github.com → click your avatar (top right corner) → **Settings**
2. Left sidebar → **SSH and GPG keys**
3. Click the green **New SSH key** button (top right of the page)
4. **Title**: something you'll recognize months from now, e.g. `MacBook personal — 2026-08`
5. **Key type**: leave as **Authentication Key**
6. **Key**: paste — it's already in your clipboard from step 2
7. Click **Add SSH key** → confirm with your password or 2FA if prompted

## 4. Authorize the key for an SSO-enforced organization (if you need one)
If the repo you need belongs to an org with SAML SSO enabled, a freshly added key works immediately for your personal repos but is blocked for org repos until you authorize it:
1. Settings → **SSH and GPG keys** → find the key you just added in the list
2. To the right of that key, click **Configure SSO**
3. Click **Authorize** next to the organization you need
4. You'll be redirected to that org's SSO login — complete it

**Why it bites:** skipping step 4 gives you a key that `ssh -T git@github.com` confirms is perfectly valid, but `git clone`/`git push` against an org repo fails with a permission-denied that looks identical to a wrong-key error. The key isn't wrong — it's just never been authorized for that specific org.

## 5. Test the connection
```
ssh -T git@github.com
```
Expect: `Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.`
