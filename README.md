# commentator

An agent skill for code comments that earn their place: what the code cannot say, contracts the signature does not carry, and effects that happen somewhere else.

Agents comment the way they talk in a session: a comment on every field, docblocks on one-line functions, types repeated in docs, and justifications written for whoever reviews the session. This skill gives them one test to apply, the patterns to remove with before and after examples, and the cases where a docblock is worth writing. It works on comments only and never changes the code they describe.

The examples use TypeScript and PHP, and the principles apply to any language.

## Install

The skill is the directory with `SKILL.md`. Clone it where your agent looks for skills:

```sh
# OpenCode
git clone https://github.com/Ozmah/commentator ~/.config/opencode/skills/commentator

# Pi
git clone https://github.com/Ozmah/commentator ~/.pi/agent/skills/commentator

# Claude Code
git clone https://github.com/Ozmah/commentator ~/.claude/skills/commentator

# Any Agent Skills client
git clone https://github.com/Ozmah/commentator ~/.agents/skills/commentator
```

OpenCode also reads `~/.claude/skills` and `~/.agents/skills`, so one of those clones serves both.

## Use

The agent loads the skill when the task matches its description: writing or editing code, or cleaning up its comments. To load it yourself:

```text
# OpenCode: mention it in the prompt
@commentator clean up the comments in src/policy.ts

# Pi: run it as a command, with the request after it
/skill:commentator clean up the comments in src/policy.ts
```

## License

[MIT](LICENSE)
