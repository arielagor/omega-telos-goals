# Reference — `telos_goals`

The plugin adds a goal graph to an Omega agent: a schema for goals and the
relations between them, a set of query rules over that graph, five skills the
LLM can call, and a prompt extension that tells it when to call them.

Loading and configuration are covered in [the README](../README.md). This page
is the API.

---

## Schema

Two atom shapes. Every relation reads **source → destination**, and the
direction matters to the rules below.

| Atom | Meaning |
|---|---|
| `(goal <id> <scope> <owner> <status>)` | A goal. `scope` is `individual` or `collective`. `status` is `active`, `proposed`, `achieved` or `abandoned`. |
| `(rel supports <src> <dst>)` | Achieving `src` advances `dst`. |
| `(rel conflicts <src> <dst>)` | `src` and `dst` cannot both be achieved. Assert once per pair; the conflict lens does not symmetrise. |
| `(rel subsumes <parent> <child>)` | `parent` is the broader goal and `child` is a sub-goal of it. |
| `(rel depends-on <src> <dst>)` | `src` cannot progress until `dst` is achieved. |

A goal id carries exactly one status. `telos-goal-add` refuses an id that is
already in the graph and points at `telos-goal-status` instead, because two
`(goal <id> ...)` atoms with different statuses would both answer the lenses —
the goal would read as achieved and still block its dependants.

---

## Skills

Each skill receives everything after its name as a single string, which is how
Omega parses a skill call (see `helper.balance_parentheses`), and reads the
arguments back into an expression itself. Arguments are validated before
anything is asserted.

### `telos-goal-add goal_id scope owner status`

```
telos-goal-add alice-train individual alice active
  -> (recorded (goal alice-train individual alice active))
telos-goal-add alice-train individual alice active
  -> (already-recorded (goal alice-train individual alice active))
telos-goal-add alice-train individual alice achieved
  -> (telos-error goal-exists (goal alice-train individual alice active) use-telos-goal-status)
telos-goal-add alice-train personal alice active
  -> (telos-error unknown-scope personal (individual collective))
telos-goal-add alice-train individual alice
  -> (telos-error expected-arguments (goal_id scope owner status) (alice-train individual alice))
```

### `telos-goal-status goal_id status`

```
telos-goal-status gpu-quota achieved
  -> (updated (goal gpu-quota collective dao achieved) was proposed)
telos-goal-status no-such-goal achieved
  -> (telos-error unknown-goal no-such-goal)
```

The goal's scope and owner are preserved; only the status atom is replaced.

### `telos-rel-add relation source_goal_id target_goal_id`

```
telos-rel-add conflicts alice-train bob-share
  -> (recorded (rel conflicts alice-train bob-share))
telos-rel-add causes alice-train bob-share
  -> (telos-error unknown-relation causes (supports conflicts subsumes depends-on))
```

### `telos-report`

The whole graph, read through every zero-arity lens at once.

### `telos-goals-owned-by owner`

The individual goals of one stakeholder, as `(goal-of <owner> <goal>)`.

---

## Lenses

These are MeTTa rules, callable from other MeTTa code and through the `metta`
skill. The zero-arity ones are non-deterministic queries; `(telos-reading)` is
their superposition and `telos-report` is the collapsed form the skill returns.

| Rule | Yields | Meaning |
|---|---|---|
| `(telos-conflicts)` | `(conflict-between $a $b)` | Every asserted conflict pair. |
| `(telos-collective-goals)` | `(collective-goal $g $owner)` | Every collective goal, any status. |
| `(telos-achieved-goals)` | `(achieved-goal $g $owner)` | Every achieved goal, any scope. |
| `(telos-abandoned-goals)` | `(abandoned-goal $g $owner)` | Every abandoned goal. Surfacing these stops the agent chasing a goal its owner dropped. |
| `(telos-subgoals)` | `(subgoal $child of $parent)` | The sub-goal structure declared by `subsumes`. |
| `(telos-blocked)` | `(blocked $g on $dep)` | A goal whose `depends-on` target is not yet achieved. An abandoned dependency still blocks: it will never become achieved. |
| `(telos-aligned)` | `(aligns $i with $c)` | An individual goal that `supports` a collective goal. |

Parameterised probes, deliberately outside the reading because they take an
argument:

| Rule | Yields |
|---|---|
| `(telos-goals-of $owner)` | `(goal-of $owner $g)` for each individual goal of `$owner`. |
| `(telos-achieved $g)` | `True` when `$g` has status `achieved`, nothing otherwise. |

Vocabulary is available as `(telos-scopes)`, `(telos-statuses)` and
`(telos-relations)`; `(telos-member $x $list)` is the deterministic membership
test the validators use.

---

## Prompt extension

`(telos-prompt-extension)` returns the `GOAL GRAPH` text installed under the
handle `telos-goals` when the plugin loads. It tells the model to record what a
message reveals about what someone wants, gives the exact call shapes, and adds
one standing instruction: report every `conflict-between` and `blocked` result
to the user before acting on a goal.

The extension names the skills rather than raw `add-atom` calls on purpose. A
MeTTa plugin is loaded into its own space (`&plugin-telos_goals`), and the
lenses query that space; an atom asserted through the `metta` skill lands in
the agent's space instead, where nothing here will find it.

---

## Worked example

```
telos-goal-add alice-train individual alice active
telos-goal-add bob-share individual bob active
telos-goal-add dao-fair-access collective dao active
telos-goal-add gpu-quota collective dao proposed
telos-rel-add conflicts alice-train bob-share
telos-rel-add supports bob-share dao-fair-access
telos-rel-add depends-on dao-fair-access gpu-quota
telos-rel-add subsumes dao-fair-access gpu-quota

telos-report
  -> ((conflict-between alice-train bob-share)
      (collective-goal dao-fair-access dao)
      (collective-goal gpu-quota dao)
      (subgoal gpu-quota of dao-fair-access)
      (blocked dao-fair-access on gpu-quota)
      (aligns bob-share with dao-fair-access))

telos-goal-status gpu-quota achieved
  -> (updated (goal gpu-quota collective dao achieved) was proposed)

telos-report
  -> ((conflict-between alice-train bob-share)
      (collective-goal dao-fair-access dao)
      (collective-goal gpu-quota dao)
      (achieved-goal gpu-quota dao)
      (subgoal gpu-quota of dao-fair-access)
      (aligns bob-share with dao-fair-access))
```

Bob's goal advances what the group wants; Alice's collides with it; the group's
goal was blocked until its prerequisite landed. The agent reads that off the
graph instead of inferring it again on every turn.
