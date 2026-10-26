# Editing these skills

Both skills are plain Markdown with no build step.

## Before sending a change

Run these two from inside the skill directory:

```sh
grep -cE '^[[:space:]]*```' SKILL.md
awk '/^[[:space:]]*```/{b=!b; next} !b' SKILL.md | sed 's/`[^`]*`//g' | grep -n '<'
```

The first number comes out even when every fenced block is closed. The second prints a line for each `<` sitting outside a fence, where the renderer reads it as an HTML tag and drops the rest of the sentence.

Then raise `metadata.version`. The frontmatter `name` has to match the directory name, or the client refuses to load the skill.

## Report what failed

Paste the error text. The 401 from the GitHub MCP endpoint carries four different bodies pointing at four different fixes, and the body is the only thing that says which one applies.

Say which client you were in and which step you were on.
