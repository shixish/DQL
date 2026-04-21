# Diffbot DQL Skill

A Claude Code skill for querying the [Diffbot Knowledge Graph](https://docs.diffbot.com/docs/getting-started-with-diffbot) using natural language. You describe what you're looking for; Claude constructs the DQL query and runs it.

## Setup

**1. Get your Diffbot API token** from https://app.diffbot.com/get-started/

**2. Store it locally** (one time per machine — never committed to the repo):

```bash
mkdir -p ~/.diffbot && chmod 700 ~/.diffbot
echo "token=YOUR_TOKEN_HERE" > ~/.diffbot/credentials
chmod 600 ~/.diffbot/credentials
```

**3. Open this project in Claude Code.** The skill is picked up automatically from `.claude/skills/dql/`.

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

The file lives at `~/.diffbot/credentials` on your local machine and is never part of this repository.
