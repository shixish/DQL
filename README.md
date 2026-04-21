# Diffbot DQL Skill

A Claude Code skill for querying the [Diffbot Knowledge Graph](https://docs.diffbot.com/docs/getting-started-with-diffbot) using natural language. You describe what you're looking for; Claude constructs the DQL query and runs it.

## Dependencies

- **`curl`** — pre-installed on macOS and most Linux distros
- **`jq`** — JSON query tool used to inspect the ontology cache
  - macOS Ventura+: ships with the OS. Otherwise: `brew install jq`
  - Linux: `sudo apt install jq` / `sudo dnf install jq`
  - Windows (WSL): `sudo apt install jq`

## Setup

**1. Get your Diffbot API token** from https://app.diffbot.com/get-started/

**2. Open this project in Claude Code** and run `/dql` once. The skill will create `~/.diffbot/` and cache the ontology automatically, then tell you credentials are missing.

**3. Store your token:**

```bash
echo "token=YOUR_TOKEN_HERE" > ~/.diffbot/credentials && chmod 600 ~/.diffbot/credentials
```

That's it. Run `/dql` again and it's ready.

## Usage

Invoke with `/dql` followed by a plain-text description:

```
/dql find large tech companies in Austin, Texas
/dql show me CTOs at public biotech companies
/dql recent negative articles about OpenAI
/dql top cities where data scientists work
/dql software startups in Berlin under 100 employees with a female CEO
```

Claude will construct the DQL query, execute it against the Diffbot API, and return formatted results. You can ask for the next page, refine the query, or request a different format.

## Credentials file format

```
token=YOUR_DIFFBOT_TOKEN_HERE
```

The file lives at `~/.diffbot/credentials` on your local machine and is never part of this repository. The ontology cache at `~/.diffbot/ontology.json` is refreshed automatically each time the skill runs.
