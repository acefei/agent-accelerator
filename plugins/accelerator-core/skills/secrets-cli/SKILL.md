---
name: secrets-cli
description: Look up credentials in the local `secrets` CLI (sops + age) BEFORE asking the user for them. Use whenever a task needs a token, password, API key, connection string or any other credential to reach a resource — a GitHub / Jira / Jenkins / Artifactory API, a database, a container registry, an SSH host, or a script that reads an env var. Also use when a command fails with 401 / 403 / "authentication required", when a `.env` value is missing, or when the user asks to store, list, search or inject a secret. Inject values with `secrets exec` so they never reach the transcript, argv or shell history. Only ask the user if the store does not have the name.
---

# Secrets CLI

`secrets` is a small wrapper over **sops + age**. One encrypted JSON file holds every
value; one age key decrypts it. This skill is the rule for using it:

> **Ask the store first. Ask the user last.**

Stopping to ask for a token that is already on the machine wastes the user's turn, and it
tempts them to paste the value into the chat, where it stays forever.

## Phase 0 — Check the store is usable

```bash
command -v secrets            # not found -> see "If the CLI is missing"
secrets list                  # prints one name per line; non-zero if key/store missing
```

`secrets list` failing means no store or no key, not "no secrets". Report which.

## Phase 1 — Find the name

Names are env-var style (`GITHUB_TOKEN`, `JIRA_API_TOKEN`). Search before you conclude
that a credential is absent — the name in the store rarely matches your first guess:

```bash
secrets list
secrets search 'jira|atlassian'    # regex over NAMES only, never values (already case-insensitive)
```

Try the obvious aliases (`GH_TOKEN` vs `GITHUB_TOKEN`) before you give up.

## Phase 2 — Use the value without exposing it

Pick the narrowest form that works.

| Situation | Command |
|---|---|
| Run one command that reads env vars | `secrets exec -- <cmd> <args>` |
| Same, but hide the rest of your environment | `secrets exec --pristine -- <cmd>` |
| Load a named subset into this shell | `eval "$(secrets export --file names.txt)"` |
| Feed one value on stdin | `secrets get NAME \| <cmd> --token-stdin` |

`secrets exec` is the default. It adds the values to the child environment only, so
nothing lands in your shell, in `ps`, or in the transcript.

**Never** do any of these, even once:

```bash
echo "$(secrets get TOKEN)"           # prints the value into the transcript
curl -H "Authorization: Bearer $(secrets get TOKEN)" ...   # visible in ps and history
secrets get TOKEN > /tmp/token        # plaintext on disk
```

Use `secrets exec -- curl -H 'Authorization: Bearer '"$TOKEN" ...` instead, quoting so the
shell expands `$TOKEN` inside the child, not before it.

## Phase 3 — Only now, ask the user

If the name is genuinely absent, tell the user exactly what is missing and what it is for.
Ask them to store it themselves, so the value never passes through you:

```bash
secrets set GITHUB_TOKEN      # prompts, input hidden, not echoed
```

Then continue with Phase 2. Do not ask them to paste the value into the chat. If they
paste one anyway, use it, tell them it is now in the transcript, and suggest they rotate it.

## If the CLI is missing

`secrets` ships in [dotfiles-mise](https://github.com/acefei/dotfiles-mise) as
`utility/secrets`. It needs `sops`, `age` and `jq` on PATH — `mise` supplies all three.
Say it is missing and let the user install it. Do not fall back to plaintext.

## Reference

| | |
|---|---|
| Store | `$SECRETS_FILE`, default `~/.config/mise/secrets.json` — encrypted, safe to commit |
| Key | `$SOPS_AGE_KEY_FILE`, default `~/.config/mise/age.txt` — **never** commit or print |
| Commands | `init`, `list`, `get`, `set`, `delete`, `search`, `export`, `exec` |

**Guardrails:** never print, log or write a secret value; never pass one as a command-line
argument; never read the age key; and never commit either file without checking which one
you have.
