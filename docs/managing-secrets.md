# Managing Secrets

How to create, edit, and work with encrypted Secrets in this repo using SOPS + age.

## Prerequisites

- `sops` and `age` installed
- The `age.agekey` file available locally
- Set the key file path so sops can decrypt:
  ```bash
  set -gx SOPS_AGE_KEY_FILE /path/to/age.agekey
  ```

## Creating a new Secret

1. Write a plain Secret manifest:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: my-secret
     namespace: my-namespace
   type: Opaque
   stringData:
     api-key: "the-actual-value"
   ```

2. Encrypt it in-place:

   ```bash
   sops --encrypt --in-place path/to/secret.yaml
   ```

   The `stringData` values are replaced with `ENC[...]` blobs. Metadata stays readable.

3. Add it to the relevant `kustomization.yaml` and commit.

## Editing an existing encrypted Secret

```bash
sops path/to/secret.yaml
```

Opens the file decrypted in `$EDITOR`. Re-encrypts automatically on save.

## Viewing an encrypted Secret

```bash
sops --decrypt path/to/secret.yaml
```

Prints the decrypted YAML to stdout without modifying the file.

## How it works

- `.sops.yaml` at the repo root tells the `sops` CLI which fields to encrypt (`data` and `stringData`) and which age public key to use. It contains no sensitive data.
- Each encrypted file has a `sops:` metadata block baked in. Flux reads this block at apply time and decrypts using the `sops-age` Secret in the `flux-system` namespace.
- Unencrypted Secrets (like ServiceAccount token Secrets with no `data`/`stringData`) are applied as-is — Flux only decrypts files that have the `sops:` metadata.

## Safeguards

A pre-commit hook and a CI check both scan for unencrypted Secrets. If a Secret has `data` or `stringData` but no `sops:` metadata, the commit (or PR) will be blocked.

To enable the pre-commit hook after cloning:

```bash
git config core.hooksPath .githooks
```

This only needs to be done once per clone. The CI check runs automatically on every PR regardless.

## Things to remember

- Always encrypt before committing. Plaintext secret values must never appear in git history.
- `age.agekey` must never be committed. It is in `.gitignore`.
- If you lose the age key, you cannot decrypt existing secrets. Keep it in your password manager.
