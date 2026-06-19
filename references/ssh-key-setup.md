# SSH Key Authentication Setup

One-time setup for passwordless SSH connections. After this, all SSH commands run without password prompts.

## Step 1: Check for Existing Local Key

```bash
cat ~/.ssh/id_ed25519.pub 2>/dev/null || cat ~/.ssh/id_rsa.pub 2>/dev/null
```

## Step 2: Generate Key if None Exists

```bash
ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)" -N "" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

## Step 3: Push Key to Server (User Enters Password Once)

```bash
ssh -p [port] [user]@[host] "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '[PUBLIC_KEY_CONTENT]' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo 'SSH key added successfully'"
```

Replace `[PUBLIC_KEY_CONTENT]` with the full public key line from step 1 or 2.

**Why not `ssh-copy-id`?** Not available on Windows. This manual method works on all platforms.

## Step 4: Verify Passwordless Connection

```bash
ssh -o BatchMode=yes -p [port] [user]@[host] "echo 'Passwordless SSH is working!'"
```

If successful, connects without asking for a password.

## Important Notes

- One key works for ALL servers — just add the same public key to each server
- The private key NEVER leaves the local machine
- If a key was already set up for another project on the same server, it will already work
- On Windows, always use the manual push method (step 3) since `ssh-copy-id` is not available
