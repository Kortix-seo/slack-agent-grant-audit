# slack-agent-grant-audit

Kortix is open source, and this is a small read-only auditor for the one question that decides whether a Slack agent gets approved: **which of its tool calls run with nobody watching?**

An agent in Slack is easy to install and hard to sign off. The bot answers in a thread your team already reads, and behind that thread it can reach your mail, your tracker and your finance system. What a reviewer needs is not a marketing page about approvals. It is the answer for the specific calls this agent will make, read from the file that governs it.

`audit.py` reads a real `kortix.yaml` (v2 or v3), resolves each tool path you list against the project's `policies` rules, the connector's own `policies`, a connector's `sensitive` flag and `policy.default_mode`, and prints the decision with the rule that made it, plus how long a held call waits (`approvals.ttl`).

```
$ python audit.py examples/slack-project/kortix.yaml --calls examples/slack-project/calls.txt --channel slack
call                       effect       decision          decided by
kortix_slack.post_message  write        require_approval  policy.default_mode risk: a write call needs approval (4h)
gmail.search_threads       read         allow             project rule #2 gmail.search_* -> always_run
gmail.send_email           write        block             project rule #1 gmail.send_email -> block
linear.list_issues         read         allow             project rule #4 linear.list_* -> always_run
linear.create_issue        write        require_approval  connector linear is sensitive: unmatched calls need approval (4h)
github.list_pull_requests  read         allow             connector github rule #1 list_* -> always_run
github.merge_pr            write        require_approval  connector github rule #2 * -> require_approval (4h)
github.delete_repo         destructive  block             project rule #3 github.delete_repo -> block
```

That is the real output of the example project in `examples/slack-project/`, a Slack bot with `policy.default_mode: risk`, a block on `gmail.send_email`, reads allowed on the tracker, and every held call expiring after `approvals.ttl: 4h`.

## What it resolves, in Kortix's own order

Kortix decides a connector call in this order, and the tool follows the same order so its answer matches what the platform will do:

1. **The agent's grant.** `agents.<name>.connectors` is `all`, `none`, or a list. Omitted means none. A call to a connector the agent cannot reach never runs, whatever the policy says.
2. **Project rules** (`policies`). A rule matches a fully qualified `<connector>.<tool>` path with `*` globs. First match wins. A rule's `conditions` must all hold, and each condition takes `arg`, `match` (a glob, or `/regex/` in slashes) and optional `negate: true`.
3. **The connector's own `policies`.** Same shape, matched on the tool name inside that connector. Project rules are checked before these, so a manifest rule can close a gap a connector leaves open. The connector's `policies` are read from `connectors[].policies`.
4. **`sensitive: true`** on a connector entry: a call that no rule matched needs approval.
5. **`policy.default_mode`.** `allow_all` (the default) runs the call. `risk` runs a read and holds a write or destructive call.

`approvals.ttl` sets the wait: an integer with `m`, `h` or `d`, clamped to 5 minutes through 30 days, and 24 hours when the key is absent or malformed. The tool prints the same value it resolves, so a reviewer sees how long a message sits unanswered.

The four blocks this order reads — the agent grant, `policies`, the connector entry and `policy.default_mode` — are the same four a change-request review of a Slack agent has to catch; [the kortix.yaml behind a Slack agent](https://agentsinslack.com/kortix-yaml-slack-agent) walks through them as they sit in one file, with the audit's output beside each.

## Use it

```bash
pip install -r requirements.txt

# every grant an agent has, no calls needed
python audit.py path/to/kortix.yaml --agents

# the calls you care about, with the effect each one has
python audit.py path/to/kortix.yaml --check gmail.send_email=write --check gmail.search_threads=read

# a calls file, one per line: path, effect, optional JSON arguments for conditions
python audit.py path/to/kortix.yaml --calls examples/slack-project/calls.txt

# machine-readable, for a change-request check
python audit.py path/to/kortix.yaml --calls calls.txt --json
```

Exit codes: `0` every call resolved and decided; `1` at least one call is undecided (an effect the tool cannot classify, or a malformed rule); `2` the manifest or the call list could not be read. A `require_approval` or `block` decision is a resolution, not a failure, so a review that holds writes still exits `0`.

The effect of a call (`read`, `write`, `destructive`) is not in the manifest. Only the connector knows it, and under `default_mode: risk` the effect is the whole decision. That is why you declare it: an unknown effect is reported as undecided rather than guessed.

## Example: the calls a Slack channel agent makes

Seven calls drawn from what an agent installed in a Slack channel actually does:
reply in a thread, search messages, search and send mail, open a tracker issue,
merge a pull request, and write to a CRM the project never declared. Save them
one to a line in `batch.txt`.

```
kortix_slack.post_message=write
kortix_slack.search_messages=read
gmail.search_threads=read
gmail.send_email=write {"to": "customer@example.com"}
linear.create_issue=write
github.merge_pr=write
salesforce.update_record=write
```

```
$ python audit.py examples/slack-project/kortix.yaml --calls batch.txt --channel slack
call                          effect  decision          decided by
kortix_slack.post_message     write   require_approval  policy.default_mode risk: a write call needs approval (4h)
kortix_slack.search_messages  read    allow             policy.default_mode risk: a read runs
gmail.search_threads          read    allow             project rule #2 gmail.search_* -> always_run
gmail.send_email              write   block             project rule #1 gmail.send_email -> block
linear.create_issue           write   require_approval  connector linear is sensitive: unmatched calls need approval (4h)
github.merge_pr               write   require_approval  connector github rule #2 * -> require_approval (4h)
salesforce.update_record      write   block             connector salesforce is not declared in kortix.yaml
```

The last line is the one to read first. `salesforce.update_record` is blocked
because the salesforce connector is not declared in the manifest at all: the
agent's grant is resolved before any policy, so a connector the project never
handed the agent cannot be reached, whatever a rule says about it. That is a
different guarantee from a rule that stops a call. The surface was never open.

## Where it does not reach

- It reads one manifest file. It does not follow `imports`, so a project that splits agents or triggers into other YAML files needs those read separately. `kortix validate` is still the gate for the file itself; it checks the schema and not `policy` or `policies`, which is exactly the gap this tool fills.
- It does not call the platform, hold a credential or run an agent. It is a static read of a file, safe in a change-request check.
- It cannot see a connector's server-side policy changes made after the manifest was written. Read the connector settings for those.

## Why a Slack agent specifically

A Slack thread is where the approval has to happen. Kortix posts a card in the thread for a held call, and the person who started the session or a project manager answers it there. If a call you thought was held is actually allowed, nobody sees a card and nobody notices. Run the audit on the calls a bot will make, not on the calls you hope it makes.

It is also the question that separates the three products a team is usually choosing between. A Slackbot, an Agentforce agent and a Slack agent on a runtime the company runs all answer in the same channel; they differ in where the work runs and in who holds the file the audit reads. [Slack agent vs Slackbot vs Agentforce](https://agentsinslack.com/slack-agent-vs-slackbot-vs-agentforce) puts the three side by side on exactly those points.

Slack now ships its own room for that work: a code channel per task, kept as a record once the work lands. [Slack Code vs a Slack agent](https://agentsinslack.com/slack-code-vs-slack-agent) reads the two records against each other: what the channel keeps, and where the file that decides which tool calls run unattended actually lives.

## Background

Kortix is the open-source CompanyOS: your agents, their skills, your company memory, and every connector in one git repo you own, with the agents working on real cloud computers. Slack is one of its channels: a message in a thread starts a session, the agent works on its own cloud computer, and the answer comes back in the same thread.

- What an agent in Slack actually is, and what connecting it takes: [agents in Slack](https://agentsinslack.com/)
- The four `kortix.yaml` blocks a Slack agent lives in, and what a change-request review of the file should catch: [the kortix.yaml behind a Slack agent](https://agentsinslack.com/kortix-yaml-slack-agent)
- Which calls stop for a person, and how the approval card behaves in a thread: [approval gates for a Slack agent](https://agentsinslack.com/slack-approval-gates)
- Running the runtime yourself: [a self-hosted Slack agent](https://agentsinslack.com/self-hosted-slack-agent)
- How the Slack app is connected to a project, step by step: [connecting Slack to a Kortix project](https://agentsinslack.com/slack-agent-setup)
- How a Slack agent differs from a Slackbot and from an Agentforce agent: [Slack agent vs Slackbot vs Agentforce](https://agentsinslack.com/slack-agent-vs-slackbot-vs-agentforce)
- The manifest reference this tool implements: [kortix.yaml reference](https://kortix.com/docs/developers#kortixyaml-reference)

## Tests

```bash
python -m unittest discover -s tests -v
```

The tests build manifests in a temp directory and assert the decision, the deciding rule and the resolved TTL for each case, including first-match-wins, globs, conditions with `negate`, an omitted grant, `sensitive: true`, a malformed TTL and a read under `default_mode: risk`.
