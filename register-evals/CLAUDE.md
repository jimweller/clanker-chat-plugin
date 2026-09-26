# Clanker Register eval suite

This suite measures whether the clanker-chat plugin makes a `claude -p` reply follow
the Clanker Register in `rules/clanker-register.md`. Each case is one operator prompt.
Two writers answer it, one with the plugin and one without, and a JSON judge grades
each reply.

## Running it

```bash
cd register-evals
npx promptfoo@0.123.1 eval --repeat 3 --no-cache -o /tmp/register.json
tools/register-report.py /tmp/register.json
node --test tests/ && python3 -m unittest discover -s tests -p 'test_*.py'
```

No install step. The writers and judges authenticate through the parent shell's
Foundry or Anthropic key and read routing from `~/.claude/settings.json`, so no API key
goes in a config.

## Writers

`providers/writer.py` runs both arms from a fresh temp directory with
`--setting-sources ""`, so neither sees this machine's CLAUDE.md, settings, hooks, or
git state. Both hold `Bash` and `Read` under `--permission-mode dontAsk`. The `with`
arm adds `--plugin-dir` on this clone, the plugin's output style, and permission to
Read the rules file its `SessionStart` hook points at. The `baseline` arm loads no
plugin. `--bare` is never used, because it disables plugin hooks.

The writer returns an `ISOLATION_BREACH` error row when the plugin loads from any path
other than this clone, a hook fails to fire, or the output style differs. The installed
copy of the plugin carries the same name, so only the path tells the two apart. Whether
the writer follows the hook's pointer and reads the rules is recorded as
`contract_loaded` and never treated as a breach.

`--setting-sources ""` also drops `modelSettings`, so the writer passes the effort a
real session uses for its model, resolved from `~/.claude/settings.json`. A model with
no effort is a writer error.

## The judge

`graders/judge.js` runs two judges on every reply, three `claude -p --bare` passes
each. The register judge grades against `prompts/register-comply-json.txt` with a JSON
schema whose rule enum is the `CR-` ids. The correctness judge checks the reply
against `prompts/correctness-judge-json.txt`. A reply is register-clean when a majority
of passes report no finding. `tools/extract-register-catalog.py` cuts the rules out of
`rules/clanker-register.md` on the `<clanker-register>` tag, and `corpus/catalog.md`
is regenerated from it on every run.

| Knob                          | Default                                   | Effect                                       |
| ----------------------------- | ----------------------------------------- | -------------------------------------------- |
| `EVAL_MODEL`                  | `opus`                                    | Writer model alias                           |
| `EVAL_EFFORT`                 | the alias's `modelSettings` `effortLevel` | Writer effort, so a run matches a real one   |
| `JUDGE_MODEL`, `JUDGE_EFFORT` | `opus`, `high`                            | Judge model alias and effort                 |
| `JUDGE_PASSES`                | 3                                         | Judge passes per reply                       |
| `JUDGE_MAX_PROCS`             | 36                                        | Concurrent judge processes per promptfoo run |
| `JUDGE_CATALOG`               | unset                                     | Pins the judge to a catalog copy for an A/B  |

Env beats config, which beats the default.

## Tools

`tools/register-report.py` prints the clean, per-pass, agreement, and correctness rates
for each arm. It recomputes every rate against promptfoo's derived metrics and exits 1
on a mismatch, a cached row, an ungraded row, a judge error, or a writer error.

`tools/rejudge-tests.py` turns a run's `-o` JSON into `corpus/rejudge/tests.json`, and
`promptfooconfig.rejudge.yaml` grades those stored replies without running a writer.
`tools/judge-agreement.py` compares passes within one run with `within`, or two runs
over the same replies with `across`.

## Cases

`cases/scenarios.csv` holds 6 prompts. Each targets one register behavior, covering a
terse question, a wrong idea to correct, an ambiguous request, a question that invites a
first-person answer, a destructive action, and a simple fact. At `--repeat 3` that is 18
replies per arm, too few for a per-rule count to mean much.

## Where the numbers stand

Measured on 2026-09-26 with the `opus` writer at xhigh and the judge at medium effort,
at `--repeat 3`.

| Arm           | Register-clean by majority | Correct by majority | Read the rules file |
| ------------- | -------------------------- | ------------------- | ------------------- |
| `with-plugin` | 10 of 18                   | 18 of 18            | 18 of 18            |
| `baseline`    | 2 of 18                    | 18 of 18            | 0 of 18             |

The `with-plugin` findings left are `CR-state-once` 3 times, `CR-no-assumed-step`
twice, `CR-ordering` twice, `CR-approval-words` once, and `CR-plain-speech` once.

## Judge calibration

The 36 replies above were rejudged four times. Each table row compares a judge with a
first `opus` medium rejudge, which cost $4.58.

| Judge                     | Clean-majority agreement | Majority-finding agreement | Cost  |
| ------------------------- | ------------------------ | -------------------------- | ----- |
| `opus` medium, second run | 35 of 36                 | 74 of 87                   | $4.51 |
| `opus` high               | 35 of 36                 | 79 of 88                   | $5.23 |
| `sonnet` medium           | 27 of 36                 | 50 of 92                   | $3.24 |

`opus` high matched the floor the second medium run set. `sonnet` fell well below it
and is not a substitute. The judge now defaults to high to match the prose suite, and
the numbers above were not rerun.

## Open

Six prompts are too few to separate a rule change from noise. More scenarios come
before any per-rule claim.
