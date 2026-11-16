# ASD-STE100 Skill for GitHub Copilot CLI — Simplified Technical English for Agent Output

A [GitHub Copilot CLI](https://github.com/features/copilot/cli) skill that rewrites dense, ambiguous English into [ASD-STE100 Simplified Technical English](https://www.asd-ste100.org/) (STE) — the controlled-language standard the aerospace and defense industry built so aircraft maintenance instructions cannot be misread.

This skill repurposes that same discipline for a different reader: an **AI agent** parsing another agent's output, a tool description, an error message, or an inter-agent instruction, with no human in the loop to resolve ambiguity.

## Credit

This repo is a Copilot CLI port of [danyuchn/asd-ste100-skill](https://github.com/danyuchn/asd-ste100-skill), created by **Dustin Yuchen Teng** ([@danyuchn](https://github.com/danyuchn)) and its contributors. The original is a Claude Code skill. All credit for the skill, the rules, the examples, and the linter goes to the original author and contributors. The upstream commit history is kept in this repo.

What changed for Copilot CLI:

- The installation steps install the skill into the Copilot CLI personal skills directory (`~/.copilot/skills/`).
- The `.claude-plugin/` manifests are removed.
- The README has an optional section that makes STE the default output style for every Copilot CLI session.

`SKILL.md`, `references/`, `examples/`, and `scripts/ste-lint.py` are the same as upstream. To report a problem with the skill itself, open an issue on the [original repo](https://github.com/danyuchn/asd-ste100-skill/issues).

## Why STE, and Why for Agents

STE exists because a misread instruction on an aircraft can kill people, and the intended readers were often not native English speakers with no author to call for clarification. The standard's fix: one meaning per word, active voice, simple tenses, one instruction per sentence, short sentences, no dropped words.

An LLM agent parsing another agent's output is in a strikingly similar position — no back-channel, no way to ask "did you mean X or Y?" The same rules that keep a mechanic from misreading a torque spec keep a downstream agent from misreading a tool description or an inter-agent message.

## Before / After

| Before | After |
|---|---|
| "This tool will attempt to synchronize state across the various backends that have been configured, and if a conflict is detected it may resolve it automatically depending on the strategy that has been set, or otherwise it will surface the conflict for manual review." | "The tool tries to synchronize state across the configured backends. If it finds a conflict, it reads the configured strategy. If the strategy allows automatic resolution, the tool may resolve the conflict without a user. If the tool does not resolve the conflict, it reports the conflict for manual review." |
| "An error may have occurred while processing your request due to a possible mismatch in the expected data format, which could be caused by an outdated client version." | "Your request may have failed. The cause may be a data format that does not match what the server expects. An outdated client can cause this mismatch. Check your client version." |

More examples, including illustrations of the official STE rules themselves, in [`examples/before-after.md`](examples/before-after.md).

## What This Skill Does

1. Picks a mode. **Strict** covers procedures, error messages, and tool descriptions. **STE-flavored** covers READMEs, PR descriptions, and explanatory prose. STE-flavored keeps the sentence discipline but not the fixed-vocabulary lockdown.
2. Reads the input English text for meaning.
3. Flags every rule violation sentence-by-sentence: ambiguous word choice, present-perfect/complex tense, passive voice with an unclear actor, multi-instruction sentences, oversized noun clusters, dropped words, sentences over length, phrasal verbs, nominalized actions, semicolons, hedge stacks, and marketing adjectives.
4. Rewrites each flagged sentence — without dropping any fact, condition, or scope qualifier from the original. If a shorter phrasing would lose required precision, it keeps the longer phrasing and flags the trade-off instead of silently simplifying.
5. Outputs the rewritten text on its own — no preamble, no mode announcement, no change summary — plus a one-line `Kept as-is:` note when it deliberately left something unsimplified.

Ask for the reasoning ("show the diff", "which rules did it break") and it outputs a before/after table naming each rule instead.

The structural rules it checks are mechanical — you can point at the word or punctuation mark that breaks each one. The rules that depend on ASD's dictionary are flagged as advisory rather than enforced, and the rules that need taste are left to you.

The linter checks structural patterns only. It does not compare an original text with a rewrite, verify that requirement strength stayed the same, or prove that the rewrite preserved meaning. A zero-violation result means that the configured structural checks found no problems.

The deterministic linter checks semicolons, a short list of soft phrasal verbs (spin up, reach out, dive into, kick off, circle back, touch base), nominalizations, marketing adjectives, passive voice, present-perfect forms (including irregular participles such as "has run"), long sentences, synonym rotation, and dangling conjunctions in supported list items. It does not check noun-cluster length (that needs part-of-speech tagging) and it does not know phrasal verbs outside its list, so "take off the panel" passes. It never flags hedges or modality.

For the sentence-length check, a word is a whitespace-separated token with at least one letter or digit. Inline code in single backticks, list markers, blockquote markers, and bare punctuation such as a spaced em dash are not words.

The repo's own prose does not lint clean: the rule tables quote the patterns they forbid, and some sentences run long. Lint it with `python scripts/ste-lint.py --baseline 36 SKILL.md` and read the findings as examples, not defects.

The dangling-conjunction rule checks list markers at the start of a line with zero to three leading spaces and ASCII spaces after the marker. It supports unordered markers `-`, `*`, and `+`, and ordered numeric markers that end in `.` or `)`, such as `1.` or `1)`. It checks indented continuation lines up to the final meaningful line. It does not parse list syntax inside blockquotes, lazy continuation, or full nested-list semantics. A standalone line with four or more leading spaces is not treated as a list marker. Within an active list item, indentation at the computed content column is treated as continuation text. Fence detection follows CommonMark for blockquotes and for the fence itself. A fence opens on three or more backticks or tildes, also inside a blockquote. It closes on a run of the same character that is at least as long and has only spaces or tabs after it. A fence in a blockquote closes when the quote ends. In an HTML block, a `>` line is text and opens no fence. An HTML block starts on a block tag such as `<div>`, or on one complete tag alone on its line that does not continue a paragraph. Inline HTML such as `<span>` in a line of text starts no HTML block. In the main pass, no fence opens in an HTML block, at any quote depth. An HTML block can also sit in a blockquote, and it ends when its quote ends. The dangling-conjunction pass also tracks HTML blocks, in a list item and on the line of a list marker too. In an HTML block, no fence opens. A line with less indent than the item ends the item and the HTML block in it. On GitHub, a `<pre>` block holds code, and a comment or a `<?`, `<!X` or CDATA block holds hidden text. If such a block spans several lines, its lines before the last line are no item text, and no list item starts there. An open comment in a list item hides the text up to `-->`, also over the next items, and the text after `-->` joins the item. GitHub does the same. Outside list items, no list item starts in any HTML block. In a list item, the text of another HTML block, such as a `<div>` block, is item text with its tags. GitHub also shows the text of a `<script>`, `<style>` or `<textarea>` block, because it escapes these tags. A list marker there starts an item, and the block goes on in that item, so a fence line in it is still text. This pass does not track an HTML block in a blockquote. The main pass accepts a fence at any indent. It also accepts a closing fence up to three columns deeper than the opening fence, also at the top level. The main pass does not detect a fence on the line of a list marker. Its closing line then opens a fence, and the main pass skips the text after it. The dangling-conjunction pass detects such a fence, also after a tab. It also detects a fence in an empty item. Such a fence ends when the item ends. After a fence on a marker line, later fences in list items also end when their items end. Before that, a fence in a list item can go on after the item ends.

The intentionally invalid examples/linter-edge-cases.md file demonstrates incomplete Markdown list items. Run python scripts/ste-lint.py examples/linter-edge-cases.md to confirm that the linter reports the two expected findings. The file is a test fixture and should not be used as compliant STE prose.

It does **not** reproduce ASD's official ~900-word approved dictionary. The standard is free to obtain but not free to redistribute: Issue 9 permits reproduction only with ASD's written authority, or by eight listed categories of organisation that this project does not belong to. This skill applies the underlying *principle* (plainest available word, used the same way every time) rather than checking against a fixed word list. For certified STE-compliant documentation, use the real standard.

Full rule summary and citations: [`references/writing-rules.md`](references/writing-rules.md).

## Installation

Copilot CLI discovers personal skills in `~/.copilot/skills/<name>/SKILL.md`. Clone this repo into that directory:

```bash
git clone https://github.com/Marcus-Lindbloom/asd-ste100-copilot-cli-skill ~/.copilot/skills/asd-ste100
```

The skill is then available in every Copilot CLI session and every project.

To install the skill for one project only, clone it into the project's `.github/skills/` directory:

```bash
git clone https://github.com/Marcus-Lindbloom/asd-ste100-copilot-cli-skill .github/skills/asd-ste100
```

### Verify

```bash
copilot skill list | grep asd-ste100
```

In an interactive session, `/skills list` or `/env` also shows the skill. Start a new session after you install the skill.

### Update

```bash
git -C ~/.copilot/skills/asd-ste100 pull
```

### Remove

```bash
rm -rf ~/.copilot/skills/asd-ste100
```

### Optional: make STE the default output style

A skill loads only when a request matches its description. To make Copilot CLI write every response in STE style, add a section to your personal instructions file, `~/.copilot/copilot-instructions.md`. Copilot CLI loads this file in every session. Check with `copilot instruction list`.

Example section:

```markdown
# Output style: ASD-STE100 (Simplified Technical English)

Write every response to the user in STE style. Also use STE style for commit
messages, PR descriptions, code comments, error messages, tool descriptions,
and prompts for sub-agents. The full rule set is in the `asd-ste100` skill at
`~/.copilot/skills/asd-ste100/SKILL.md`. Load that skill when you rewrite text
or when you write text that another agent or tool will parse.

Modes:
- Chat responses, READMEs, PR descriptions, and docs use STE-flavored mode.
  Apply the structural rules in full. Treat the lexical rules as advisory.
- Error messages, tool descriptions, sub-agent prompts, procedures, and safety
  text use Strict mode. Apply all rules.

Structural rules (always apply):
1. Use active voice. Name the actor.
2. Write one instruction per sentence.
3. Keep instructions at 20 words or fewer. Keep descriptions at 25 words or fewer.
4. Do not use semicolons. Write separate sentences.
5. Do not use phrasal verbs ("spin up", "kick off", "reach out"). Use one plain verb.
6. Do not stack more than 3 nouns as one noun phrase.
7. Do not drop the subject, verb, or article to save space.
8. Use simple tenses. Keep a compound tense only when it carries information,
   for example "may have failed" or "has completed" (output available now).
9. Write one topic per paragraph, with 6 sentences or fewer.
10. Use a numbered or bulleted list for 3 or more steps or conditions.
11. Use the verb, not the noun form ("analyze", not "perform an analysis").
12. Use one name for one thing in a response. Do not rotate synonyms.
13. Do not use marketing adjectives (seamless, robust, powerful).
14. Do not stack hedges. State the claim, or delete it.

Content rules (always apply):
- Keep every hedge at its original strength. Do not change "may" to "will".
- Do not add a fact, cause, or certainty that the evidence does not support.
- Keep safety conditions, scope qualifiers, and numbers, even if the sentence
  becomes longer.
- Do not apply STE to code, command output, quoted text, or file content that
  must stay verbatim.
```

The 14 structural rules repeat the rules in `SKILL.md` as a summary, so Copilot can apply them without loading the skill first.

## Usage

Trigger with a request to simplify or clarify English text:

```
Disambiguate this tool description
Rewrite this error message so an agent can't misparse it
Apply ASD-STE100 to this instruction
```

Or paste text and ask Copilot to "disambiguate this" / "apply STE100 to this" / "reduce ambiguity in this output." You can also run `/asd-ste100` in an interactive session.

You get the rewritten text back and nothing else. To see which rules were applied, add "show the diff" or "explain the changes" to the request.

To run the linter directly:

```bash
python3 ~/.copilot/skills/asd-ste100/scripts/ste-lint.py path/to/file.md
python3 ~/.copilot/skills/asd-ste100/scripts/ste-lint.py --selftest
```

## Scope

Built for: agent-to-agent messages, tool/function descriptions, error messages, system prompts, inter-agent instructions — any English text a machine or non-native reader has to parse without a human to ask.

Not built for: creative writing, marketing copy, or anything where voice and nuance are the point — STE is deliberately flat and literal by design.

One limit worth stating up front: this fixes the form of a text, not its substance. A paragraph with nothing to say comes out short, clean, and still empty.

## Other Languages

- **Hebrew:** [g4lb/asd-ste100-hebrew-skill](https://github.com/g4lb/asd-ste100-hebrew-skill) is a Hebrew port of this skill. It includes a Hebrew linter. ASD-STE100 defines English only, so the Hebrew rules are an adaptation, not an official standard.
- **Indonesian:** [jshsinaga/asd-ste100-indonesian-skill](https://github.com/jshsinaga/asd-ste100-indonesian-skill) adapts this skill for Indonesian technical text. Its Indonesian rules are an adaptation, not an official ASD-STE100 standard.

## Related Tools

- [meiluosi/rewrite-fidelity-skill](https://github.com/meiluosi/rewrite-fidelity-skill) compares the original text with the rewrite. It flags changes in meaning and modality. It uses pattern matching, so check each flag yourself.

## Sources

- [ASD-STE100 official site](https://www.asd-ste100.org/)
- [ASD-STE100 — About STE](https://www.asd-ste100.org/about_STE.html)
- [ASD Europe — Simplified Technical English](https://www.asd-europe.org/standards-specifications/simplified-technical-english/)
- [Simplified Technical English — Wikipedia](https://en.wikipedia.org/wiki/Simplified_Technical_English)
- [TechScribe — ASD-STE100 Simplified Technical English](https://www.techscribe.co.uk/techw/asd-simplified-technical-english.htm)

## License

MIT — see [LICENSE](LICENSE). The original copyright notice by Dustin Yuchen Teng is kept, as the MIT license requires.
