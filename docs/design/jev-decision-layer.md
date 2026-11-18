# Jev as the game's decision layer — Design review

**Status:** Design-only proposal (no code written). Written 2026-09-22
after a full read of every prompt template and every place code turns a
referee word into a number. Read alongside
[architecture.md](../architecture.md) (the referee philosophy) and
[tuning.md](../tuning.md) (every knob this doc proposes to delete or
demote). Category 3 in [docs/README.md](../README.md): owes nothing to
the code until a plan doc is cut from it.

> Today's contract is **the LLM picks words; code owns numbers.** The
> proposal is a three-way split: **the LLM writes, Jev decides, code
> stores and routes.** The word ladders stay as vocabulary. The
> arithmetic that walks them (`IMPACT_STEPS`, `RESOURCE_DELTAS`, growth
> percents, step counters, event thresholds) goes away, because Jev can
> be asked for the *resulting* word directly.

---

## 1. The pitch in one breath

Every referee call in this game does two jobs in one breath: it
*decides* (impact, cost, outcome, feasibility, who acts next) and it
*writes* (the narration). The decision is squeezed out of a
text-generating model through a JSON parser, then code does arithmetic
on it: "heavy" means two rungs down the condition ladder, "notable"
means five percent on a stat, three resolved events before a goal may
complete. Jev is a model that does only the first job, natively, in
under half a second, for a few hundredths of a cent, with a
probability attached to every answer and no parser in the loop.

That changes three things:

1. **Decisions leave the prose.** Jev rules first; the LLM narrates a
   ruling it is handed. Prompts stop saying "decide first, then write to
   match" and hoping.
2. **The ladder arithmetic dies.** Instead of "impact: heavy" plus code
   stepping `wounded` two rungs to `critical`, Jev is asked "given all
   this, what is the target's condition now?" over the six ladder words
   and code stores the answer. Same for reserves, affinity, growth, goal
   progress, battle end. No addition, no thresholds.
3. **Cheap parallel judgement unlocks features** that were never
   affordable at one LLM call each: per-turn morale for every combatant,
   sampled (not argmaxed) outcomes so persuasion has real odds, input
   guardrails on player text, output QA on every narration, memory
   triage, relevance selection for context.

Jev is a week old and in early access. Section 8 says what to measure
before believing any of this.

---

## 2. What Jev actually is (verified 2026-09-22)

Jev is TypeSafe AI's "System One" model, released in limited early
access on 15 September 2026. It never writes a word. You send it
**state** (a string, a JSON object, or a list of texts) plus one or more
**typed questions**, and it returns a typed answer per question with a
probability distribution.

| Fact | Value |
|---|---|
| Endpoint | `POST https://api.typesafe.ai/v1/systemone`, bearer key; Python SDK `typesafe-sdk`, env `TYPESAFE_API_KEY` |
| Model id | `jev-1.13.0` (aliases `jev-latest`, `jev-preview`) |
| Question types | `choice` (pick one of N), `score` (position on an ordered rubric), `noul` (probability a statement is true) |
| Options per choice | **255 maximum**; above that, a rank-then-recheck pair of calls |
| Levels per score | 2 to 10, ordered, each a description ("describe situations, not degrees") |
| Context | 64K tokens per request; 32K for state plus the longest question |
| Questions per call | Many, "evaluated in parallel and in isolation against the same state"; adding questions barely changes latency |
| Latency | 70 to 500 ms end to end |
| Price | $0.042 per million input tokens; output free |
| Rate limits | 250K tokens/s, 1,200 requests/min (dynamic) |
| Input | Text only, English best |
| Key | Self-serve from console.typesafe.ai/keys |

Every answer carries `probabilities` (a full distribution over the
options or levels), `confidence` (0 to 1: how concentrated that
distribution is), and for `score` a continuous position (a 6-level
ladder can answer 2.4, meaning "between the third and fourth rung,
leaning third").

Options and levels can be objects, not just words: `{"what": ...,
"not_for": ..., "examples": [...]}` for a choice option,
`{"summary": ..., "signals": [...]}` for a score level. Instructions can
also be JSON. That is where a monster's "Responds well to / poorly to"
rubric, the referee mood, or a rule of the world goes.

**Correcting one assumption.** Jev is not "as many options as words in
the dictionary." The hard cap is 255 options per choice question. What
makes it feel unbounded is that one call can carry dozens of questions
against the same state for the price of one, and TypeSafe's own
cookbooks handle 182 and 218 candidates with a rank-then-recheck pair.
So "tons of pre-enumerated decisions" is right, as long as each single
decision has at most 255 answers, or is split into a taxonomy walk.

### The documented weak spots (jev-1.13)

TypeSafe publishes a "model jaggedness" page. The ones that bite here:

1. **It cannot count or do arithmetic.** "Does not count reliably";
   dates are read as text. Anything numeric must arrive as a word, or
   stay in code.
2. **It reads literally.** It answers the question you wrote, not the
   one you meant. Double negatives and indirection degrade it.
3. **Big irrelevant state hurts.** Accuracy falls as unrelated content
   grows. Send the slice of state the decision needs, not the whole
   dungeon log. (This cuts against the 1M-context habit: Jev gets a
   32K slice, chosen by code.)
4. **Not adversarially robust by default.** Injected instructions in
   state can sway it unless criteria say so explicitly.
5. **No rationale.** A number, never a why. The dev log will show
   distributions, not reasons.
6. **Benchmarks are vendor-run.** 67.8 percent agreement with frontier
   LLMs on TypeSafe's own four workflows; independent reviewers note the
   reference labels were model-generated. Every decision moved to Jev
   is something to measure on this game's own logs first (section 8).

### Patterns TypeSafe recommends (and this doc leans on)

- **Fan-out**: ask every question you might need in one call, including
  speculative ones; code picks what matters.
- **Confidence-gated routing**: the answer says what, confidence says
  whether to act. Below a threshold, fall back (here: to the LLM, or to
  "continue").
- **Composite scoring**: atomic scores combined in code.
- **Guardrails**: screen LLM inputs and outputs with nouls.
- **Rank then recheck**: for high cardinality, a wide cheap pass, then a
  narrow pass over the top few with full descriptions.

---

## 3. Every decision the game asks an LLM to make, stripped of prose

Read from the 57 templates in `backend/ai/llm/prompts/*.json` (two,
`door_choices` and `generate_initial_abilities`, have no caller and are
dead). "Prose"
fields (narration, dialogue, descriptions, names, backstories) are
listed only to show what stays with the LLM. "Math after" is what code
does with the word today.

### 3.1 Battle (`battle_generation.json`, `game/battle/`)

| Decision | Template | Answer space today | Math after | Jev shape |
|---|---|---|---|---|
| Who acts next | `next_turn` | one combatant name | fairness force-pick (`OVERDUE_WAIT_MULTIPLIER`) skips the LLM entirely when anyone is overdue; softlock valve (`MAX_CONSECUTIVE_ENEMY_TURNS`); `turn/director.py:36-76` | `choice` over living combatants (≤8) |
| Enemy action | `enemy_turn` | attack / ability / defend / talk / flee, + target, + ability name | none | `choice` action, `choice` target (roster), `choice` ability (its list) |
| Wary ally action | `ally_autonomous_turn` | attack / ability / defend, + target, + ability | none | same |
| Impact on target | `action_resolution`, `freeform_action_resolution` | none / light / heavy / devastating / heal_light / heal_major | `IMPACT_STEPS` walks `CONDITION_LADDER` in `manager.py:141-161`, minus one step if the target is defending | **`choice` over the six condition words: the target's NEW condition** |
| Actor exertion | same | stamina and mana each: none / minor / moderate / heavy / restore_minor / restore_major | `RESOURCE_DELTAS` walks `RESOURCE_LADDER` (`manager.py:167-187`); when the referee is silent, `turn/context.py:117` invents a default cost from `ABILITY_POOL_BY_TYPE` | `choice` over the five reserve words, once per pool: the NEW reserve |
| Feasibility of typed action | `freeform_action_resolution` | possible true/false | wasted turn on false | `noul` "could this monster plausibly do this now" |
| Impact target | same | a name or null | none | `choice` over combatants + "nobody" |
| Negotiation outcome | `battle_talk` | continue / enemies_join / enemies_yield / enemies_flee / party_spared | none | `choice`, with per-monster "Responds well/poorly to" as the criteria object |
| Battle over? | (code) `manager.py:305` `derive_outcome` | all of one side incapacitated or fled | roster check | `choice` continues / party_victory / party_defeat (Jev reads a roster; it does not count) |

Prose that stays with the LLM: `battle_arrival`, `battle_intro`,
`turn_vanity`, `battle_summary`, `battle_victory`, `battle_defeat`, the
`narration` field of both referees, the `response`/`dialogue` fields.

### 3.2 Dungeon and exploration (`dungeon_generation.json`, `exploration_generation.json`, `game/dungeon/`)

| Decision | Template | Answer space today | Math after | Jev shape |
|---|---|---|---|---|
| Goal progress | `goal_check` | no / progress / complete | `GOAL_MIN_EVENTS` counter gate, `GOAL_CHECK_LOG_TAIL` slice (`goal.py:130`) | `score` over the three levels, read the continuous position; confidence gate replaces the counter (see 5.3) |
| Sneak success | `sneak_attempt` | true/false | none | `noul`, **sampled** from its probability, not thresholded |
| Ability use outside battle | `dungeon_ability_use` | none / heal_light / heal_major / reveal + costs | `IMPACT_STEPS`, `RESOURCE_DELTAS` again (`handlers/items_abilities.py:74-130`) | `choice` new condition, `choice` new reserves, `noul` "revealed something true" |
| Item use outside battle | `dungeon_item_use` | none / heal_light / heal_major / reveal | same | same minus costs |
| Path event | (code) `events.py:45` | explore / dialogue / battle / treasure / returning | `EVENT_WEIGHTS` + danger profile, `random.choices` | keep the roll, or `choice` from theme + journal and **sample from Jev's distribution** (themed randomness) |
| Monsters present at explore | (code) `events.py:68` | true/false | `EXPLORE_MONSTERS_CHANCE` | same treatment |
| Danger word per notice | (code) `handlers/notices.py` | calm / risky / perilous | `NOTICE_DANGER_WEIGHTS` roll, then `DANGER_PROFILES` maps to five numbers | `score` over the pitch text, sampled |
| Camp restore | `camp_restore` | per monster, per pool: none / restore_minor / restore_major | `RESOURCE_DELTAS` (`handlers/camp.py:73`) | `choice` new reserve word per monster per pool, one fan-out call |

Prose that stays: `entry_atmosphere`, `random_location`, `door_choices`,
`path_choices`, `exit_path`, `arrival_location`, `opening_scene`,
`run_goal`, `expedition_notices` (title/pitch/theme), `exit_narrative`,
`look_around`, `camp_scene`, `ambush_intro`, `treasure_discovery`, every
`narration`.

### 3.3 Encounters and chat (`encounter_generation.json`, `chat_generation.json`)

| Decision | Template | Answer space today | Math after | Jev shape |
|---|---|---|---|---|
| Dialogue outcome | `monster_dialogue_turn` | continue_dialogue / begin_battle / allow_passage / reward / punish / join_party | none | `choice`; `join_party` probability **sampled** with the monster's rubric as criteria (fixes "persuasion too easy", see 5.1) |
| Secret earned? | same (buried in instruction 3) | implicit | none | `noul` "has the party earned the secret in this conversation" |
| Memory worth keeping | `chat_memory_extraction` | 0 to 3 memories, each kind ∈ learned_fact / confided / grew_closer / shared_lore / voiced_wish | `extract_after_messages` counter | `noul` triage "does this stretch hold anything with lasting weight" before the LLM call; `choice` kind per extracted memory |

Prose that stays: `home_chat_reply`, `monster_question`, the
`response`, every memory's `content`.

### 3.4 Memory, growth, returning, evolution (`memory_generation.json`, `evolution_generation.json`, `game/memory/`, `game/monster/`)

| Decision | Template | Answer space today | Math after | Jev shape |
|---|---|---|---|---|
| Camp spotlight | `camp_spotlight` | 1 to 2 party names | `SPOTLIGHT_CAP` | `score` "how much story did this monster live this run" per member, one fan-out call; code takes the top two |
| Growth stat and tier | `growth_reflection` | stat ∈ health/attack/defense/speed/none; tier ∈ slight/notable | `GROWTH_STAT_TIERS` → percent, `LIFETIME_GROWTH_CAP` (`growth.py:84-87`) | `choice` stat; `choice` tier. The percent math stays **unless** stats become word ladders (6.3) |
| New ability earned | `growth_reflection`, `evolution_abilities`, `returning_transform` | yes/no | `MAX_ABILITIES` cap | `noul` |
| Which ability to reword | same | one ability name or none | `ABILITY_REWORD_CAP`, `REWORD_MAX_RATIO` | `choice` over its abilities + none |
| Returning disposition | `returning_transform` | hostile / friendly / wary | none | `choice` |
| Returning training | same | slight / notable / fierce | `RETURN_STAT_TIERS`, multiplier step and caps | `choice`; percent math stays until 6.3 |
| Affinity step | (code) `affinity.py:67` | +1 on a fixed list of code-visible events | `MAX_AFFINITY_STEPS_PER_RUN` budget | `choice` over the four affinity words after any event: the NEW tier, confidence-gated (5.4) |

Prose that stays: reflections, memory notes, greetings, grudge notes,
battle lines, every evolution field, the reworded descriptions.

### 3.5 Generation-time snapping (`monster_generation.json`, `player_generation.json`, `ability_generation.json`)

These are mostly prose, but several fields are picks from curated enums
in `game/monster/cmdts_data.py`: party role (6), size class (6), sapience (4), communication (5), social structure (5), lifecycle
(5), habitat domain (4), feeding style (6), activity cycle (4), plus
elements, biomes, sustenance, creation mechanisms, and the taxonomy
lineage. The player blueprint's job is literally "snap the player's
chosen kind onto the curated taxonomy." Every one of these is a Jev
`choice` over the enum, made in one call against the prose the LLM
already wrote. Ability `type` (attack / defense / movement / support /
special / utility, which decides the pool via `ABILITY_POOL_BY_TYPE`)
is the same.

Two things are deliberately *not* asked of the LLM today and should
stay that way: rarity is rolled in code (`cmdts_data.py:169`, because
"a creative model asked to choose rarity drifts toward legendary") and
stats are derived from role, rarity and size (`cmdts_data.py:201`).
A Jev `score` for rarity from the finished prose would be a fair
experiment, since Jev has no taste for legendary, but it is not needed.

### 3.6 The count

Roughly **34 distinct decisions** across the 55 templates, versus about
20 pure-prose fields. Nearly every decision already has a closed answer
space of 2 to 8 words. Only three would need Jev's cardinality headroom:
target selection (a roster), species snapping (taxonomy lists), and
relevance selection over memories (up to a few hundred). None exceeds
255 without the rank-then-recheck pattern.

---

## 4. The three structural changes

### 4.1 A third generation kind in the gateway

`backend/ai/gateway.py` has `text_generation_request` and
`image_generation_request`. Add `decision_request(state, questions,
decision_name)` beside them, honoring hard rule 2 (all generation goes
through the gateway) and the log-what-the-model-saw rule:

- A `generation_log` row with `generation_type='decision'` and a
  sibling `decision_logs` table (state JSON, questions JSON, answers
  JSON with the full probability distributions, confidence per
  question, input tokens, latency). The Developer screen's AI log gains
  a distribution view: for the first time the log can show *how sure*
  the referee was.
- Provider settings: a `decision_provider` row in `game_settings`
  next to the DeepSeek row (key write-only, masked to last four), and
  an in-panel toggle. Jev off means every decision falls back to the
  existing LLM template. That fallback is the migration safety net
  and stays until the LLM decision fields are deleted.
- **Not queued.** The AI queue exists to serialize one local GPU. Jev
  calls are 100 ms HTTP requests that can run in parallel with the
  narration call; they bypass the worker and use the SDK's retry
  policy (429/529 exponential backoff).
- A question catalog, `backend/ai/decisions/*.json`, mirroring the
  prompt JSON files: one file per domain, each entry a named set of
  questions (type, instructions, criteria) plus the name of a Python
  **state slicer** in `game/**` that builds the ≤32K state object for
  it. State slicing is the new discipline: `context_limits.py` budgets
  for cost; slicers budget for relevance (jaggedness item 3).
- Offline suites: `tools/playtest/stub_llm.py` grows a `StubJev`
  twin that answers from the catalog's option lists (happy / broken /
  chaos modes, same as today). `test_step_contract.py` is untouched:
  decisions add no workflow steps.

### 4.2 Decide, then narrate

Every referee template becomes two calls in the same workflow step:

```
state = slice_for('battle_action', ctx)          # code
verdict = decision_request(state, ACTION_QUESTIONS)   # Jev, ~200 ms
narration = text_generation_request(              # LLM, streamed
    build_prompt('action_narration', verdict=verdict, ...))
```

The narration prompt receives the verdict as fact ("the blow leaves
Thornback critical; the effort leaves her strained") and writes to it.
Consequences:

- The JSON parser leaves the decision path. Missing-field and
  invalid-word fallbacks in `battle/generator.py:176,282` and every
  `word if word in RESOURCE_DELTAS else None` guard become dead code.
- Narration can stream the instant the verdict lands; today the
  player waits for the whole JSON object.
- The narration prompt shrinks: the six impact bullets, the cost rules,
  the "decide first then write to match" instruction all move into
  Jev's criteria, where they are enforced rather than requested.
- `docs/prompt-review.md` findings 1.1 and 1.2 (player text carrying
  rules authority) are solved structurally: the guardrail nouls in 5.2
  run before the referee ever sees the text.

### 4.3 Ladders as vocabulary, not as arithmetic

This is the change the request is really about. Today
`CONDITION_LADDER` is a list and `IMPACT_STEPS` is a step table; code
does `index + steps`, clamps, and stores. The proposal: Jev is handed
the ladder as the option list, the current rung as state, and asked for
the rung *after* this action. Code stores the word. Deleted:
`IMPACT_STEPS`, `RESOURCE_DELTAS`, the index math and the defend
valve in `battle/manager.py:141-187`, the silent-referee cost defaults
in `turn/context.py:117-129`, `handlers/items_abilities.py:80-82,117`,
`handlers/camp.py:73`, the `steps_delta` bookkeeping in
`turn/context.py:105-110`. Kept: the ladder lists themselves, as the
enumerated answer spaces, and `INCAPACITATED` / `SPENT` as the words
that mean "out."

Why this is safe with a model that cannot count: it is not being asked
to count. It reads "condition: wounded" and a description of a
devastating blow and picks `critical` or `incapacitated` the way a human
referee would. The ladder words are situations, which is exactly what
TypeSafe's score-level guidance asks for. The one thing to encode in
criteria is direction: a hit never improves condition, a heal never
worsens it. That can also be a code invariant (reject an answer that
moves the wrong way, keep the current rung) without being arithmetic.

The same move, ladder by ladder:

| Ladder | Today's math | After |
|---|---|---|
| Condition (6) | `IMPACT_STEPS` | Jev picks the new rung |
| Reserves (5 × 2 pools) | `RESOURCE_DELTAS` | Jev picks the new rung per pool |
| Affinity (4) | +1 per listed event, per-run budget of 2 | Jev picks the new tier after any event, confidence-gated; the budget becomes a confidence threshold (5.4) |
| Goal (3) | counter gate of 3 events | `score` position + confidence gate (5.3) |
| Danger (3) | weighted roll → five numeric knobs | Jev scores the pitch; the five knobs remain the last numbers until path events are Jev-chosen too (3.2) |
| Growth tiers (2) → percents | `GROWTH_STAT_TIERS`, caps | stays numeric unless stats themselves become ladders (6.3) |

**What still has to be a number.** Three things, and it is worth being
honest about them:

- **Stats.** Health/attack/defense/speed are integers with percent
  growth and caps. They exist to be shown on cards and read by prompts
  as magnitude. Section 6.3 sketches turning them into word ladders
  too; until then growth math stays.
- **Confidence thresholds.** Every Jev answer comes with a probability.
  "Act on `join_party` only above 0.6" is a number in code. It is not
  game math (no addition, no state stepping); it is a policy knob, and
  it lives in `tuning.md` like any other knob. This is the honest
  replacement for the valves: instead of "at most 2 affinity steps per
  run," "step affinity only when Jev is at least 0.75 sure the moment
  earned it."
- **Softlock and fairness valves.** `MAX_CONSECUTIVE_ENEMY_TURNS` and
  `OVERDUE_WAIT_MULTIPLIER` exist so a player is never locked out of
  acting and no monster is forgotten. They can become Jev questions
  ("has this monster been neglected?"), but Jev counts poorly and these
  are safety, not feel. Recommendation: keep them as the two surviving
  counters, or pass the wait as a word ("has waited a long time") and
  let Jev's `choice` for next turn carry a "neglected" criterion. Try
  the second; keep the first as a backstop for one soak.

---

## 5. New features Jev makes affordable

Each of these costs one Jev call (a fraction of a cent, well under a
second) where an LLM call would have cost a DeepSeek round trip.

### 5.1 Dice with judgement: sample, don't argmax

The live playthrough found persuasion too easy: an LLM asked "does the
monster join?" tends to say yes when the words are nice. Jev returns
`P(join_party) = 0.23`. Roll against it. The monster joins one time in
four on that exact speech, and the odds move with the rubric and the
battle state. The same trick gives sneak attempts real tension, makes
`devastating` rare by distribution rather than by prompt pleading, and
gives goal completion an earned feel. Code owns the roll (a random
number against a probability is not game math in the sense this
document retires: nothing accumulates, no threshold triggers).

### 5.2 Guardrails on player text, QA on model text

One fan-out call before the referee, over the player's typed action or
speech (`PLAYER_TEXT_MAX_CHARS` is 500, trivially within 32K):

- `noul` "contains instructions addressed to the game or referee rather
  than words spoken in the world" (the OOC injection from
  prompt-review 1.1/1.2)
- `noul` "claims a mechanical outcome (damage, cost, success) for
  itself"
- `noul` "asks the monster to do something wildly beyond its body or
  abilities" (feasibility, moved out of the referee)

And one after every narration:

- `noul` "the narration contradicts the verdict" (impact said light,
  text says the foe collapses)
- `noul` "the text is a refusal or meta-commentary rather than story"
  (the silent 500-character refusal bug from the August playthrough,
  caught before it reaches the log)
- `noul` "a character's pronouns changed from the roster" (the pronoun
  drift fix, now checked instead of begged for in every template)
- `noul` "reuses imagery or sentences from the recent log"

Fail → regenerate once with the finding in the prompt, then fall
through. This is TypeSafe's own `llm_guardrails` cookbook shape.

### 5.3 Goal progress as a position, not a counter

`goal_check` becomes a `score` over no / progress / complete. The
continuous score (1.6 = "solid progress, not there") and confidence
replace `GOAL_MIN_EVENTS`: `complete` is honored only above a
confidence threshold, and the note-taking LLM call happens only when
the position moved. The "no first-door wins" valve survives as
"complete requires confidence ≥ 0.8," which is a policy knob rather than
an event count.

### 5.4 Affinity that moves when it should

Today affinity steps only on six hardcoded events. With a `choice` over
the four tiers after every resolved event (battle action involving the
monster, dialogue turn, camp scene, chat stretch), trust can deepen
because of *what happened* rather than *which handler ran*. The per-run
budget becomes a confidence threshold, and a wary monster that watched
the player take a hit for it can become familiar that same turn. It
can also, for the first time, go *down*: a devoted monster ordered into
something cruel. That is a new mechanic and a design decision to make
deliberately, not a side effect.

### 5.5 Per-turn morale for the whole battlefield

One call per turn, fan-out: for each living combatant, `score` morale
(steadfast / pressing / wavering / desperate / broken) and `noul`
"would flee if it could." Feeds the enemy-turn `choice` as criteria,
feeds the UI (a live mood chip per card, which the August review asked
for in place of static numbers), and gives the negotiation `choice` a
real reason to say `enemies_yield`.

### 5.6 Relevance selection for context

Memories, journal lines, and recent log entries are today clamped by
token cap and recency. A `choice` (up to 255 candidates) or per-item
`noul` "bears on the current moment" lets code send the *relevant*
twelve memories instead of the *latest* twelve. Better prose, fewer
tokens, and it directly addresses Jev's own big-state weakness for the
state slicers of 4.1.

### 5.7 Memory triage

`chat_memory_extraction` runs an LLM pass every 8 unreviewed lines and
usually finds nothing ("most stretches deserve NO memory"). A `noul`
first, and the LLM only when it says yes. Same for growth reflections:
`score` "how much did this monster's journal actually show" before
spending a reflection call.

### 5.8 The roadmap items are already Jev-shaped

[roadmap.md](../roadmap.md) sketched Monster Requests (request-type
enum, `whim → wish → need → vow` weight ladder, an `unaddressed /
progressed / fulfilled / betrayed` judge), Nemesis Arcs (`brooding →
… → legend`, one upgrade from a word menu), and Party Bonds (a pairwise
ladder moved ±1 per run by an LLM proposal). Every one of those judges
and ladders is a `choice` or `score`, and every "code clamps movement to
±1" valve is exactly the arithmetic this doc retires. Bonds in
particular: a pairwise `score` over every party pair in one fan-out
call per run is affordable; it was not when each pair was an LLM call.

### 5.9 Generation-time snapping and variety

The CMDTS enums (3.5) snap in one call; `player_blueprint` stops
needing the LLM to know the taxonomy. Variety metrics from the playtest
harness could use `noul` "this monster is a near duplicate of one
already in the library" at birth, cheaply, on every generation.

---

## 6. What stays with the LLM, and two bigger ideas

### 6.1 Stays with the LLM

Anything whose answer space is not known up front: names, descriptions,
backstories, dialogue, narration, goals, notices, item text, memory
content, reflections, chronicles, summaries. Jev "does not generate
text"; forcing it to (255 pre-written narrations) would be worse than
the current design. The LLM also keeps generation-time creativity in
full; Jev snaps the *categorical* fields afterward.

### 6.2 The 32K slice versus the 1M window

The cloud-generation initiative locked a 1M-token context floor so
prompts never truncate. Jev gets 32K and gets *worse* with irrelevant
state. This is not a conflict, but it is a new job: every decision
needs a code-owned state slicer (4.1), and 5.6 is how the slicers get
smart. The LLM narration prompts keep their budgets untouched.

### 6.3 Stats as ladders (optional, later)

If numbers must leave the game entirely, health/attack/defense/speed
become five-rung word ladders (`frail → sturdy → strong → mighty →
titanic`, or per-stat vocab), growth becomes "Jev picks the new rung,"
returning boosts and evolution stage boosts become the same question,
and every cap becomes a confidence threshold or a ceiling word. Cards
show the word. Prompts already read stats only as magnitude. This is a
separate initiative with its own plan doc; it touches the monster
model, the card UI, growth, returning, and evolution.

---

## 7. What it costs

A battle turn today: about three LLM calls (director, action, referee)
at a few thousand input tokens each on DeepSeek. With Jev: the same
narration calls minus their decision fields, plus two or three Jev
calls of 2 to 4K tokens. At $0.042 per million tokens, a full battle of
twenty turns is roughly a quarter of a cent of Jev. The morale fan-out
(5.5) and QA nouls (5.2) on every turn roughly double that. The whole
overnight playtest harness could run its decision legs against Jev for
under a dollar; TypeSafe's Doom demo runs ten queries a second for about
seven dollars an hour.

Latency improves: the verdict lands in ~200 ms and narration streams
immediately, instead of waiting for a complete JSON object.

---

## 8. What to measure before believing any of it (Jev-M0)

`llm_logs` keeps every prompt byte-exact and the raw response. That is a
free evaluation set: replay logged `action_resolution`, `battle_talk`,
`monster_dialogue_turn`, `goal_check`, and `sneak_attempt` prompts
through Jev with the questions from 3.1 to 3.3 and compare Jev's
argmax against the LLM's word. Cost: cents. Output: an agreement rate
per decision, a confidence histogram, and the disagreements to read by
hand (the interesting ones are where Jev is *right* and the LLM was
gamed). This is a `tools/playtest/` script in the harness's style,
spending real provider calls and saying so.

Decision rule for going further: agreement above ~80 percent on
impact/cost with a sane confidence spread, and Jev's disagreements
reading as defensible, means 4.2 and 4.3 go into a plan doc. Below
that, Jev stays as guardrails and triage (5.2, 5.7), which need no
agreement with the old referee at all.

Unknowns to close in M0: whether an early-access key can be obtained
now; real latency from this machine; whether the 32K slice of a late
battle (roster + last dozen log lines) fits comfortably (it should, at
under 5K); how Jev handles the "Responds well to / poorly to" rubric as
a criteria object.

---

## 9. Milestone sketch (for the plan doc, if M0 passes)

- **Jev-M0** — replay experiment above; write up agreement per decision.
- **Jev-M1** — gateway `decision_request`, `decision_logs`, settings row
  and panel toggle, `StubJev`, question catalog with slicers for the
  battle referee only. LLM fallback when off. All seven checks green.
- **Jev-M2** — decide-then-narrate for `action_resolution`,
  `freeform_action_resolution`, `battle_talk`, `next_turn`,
  `enemy_turn`, `ally_autonomous_turn`. Narration templates lose their
  decision fields. Guardrail and QA nouls (5.2).
- **Jev-M3** — ladders as vocabulary: delete `IMPACT_STEPS` and
  `RESOURCE_DELTAS` and every index walk; dungeon referees, camp
  restore, goal score, affinity choice. `tuning.md` gains a "confidence
  thresholds" table and loses the step tables.
- **Jev-M4** — fan-out features: morale, relevance selection, memory
  triage, sampled outcomes. Playtest gauntlets extended with a Jev leg.
- **Jev-M5** — generation-time snapping and the roadmap judges as they
  ship.

Deviations, as always, get logged in the plan doc.

---

## Sources

- [TypeSafe docs: introduction](https://docs.typesafe.ai/introduction), [API](https://docs.typesafe.ai/api), [models](https://docs.typesafe.ai/models), [primitives: choice](https://docs.typesafe.ai/primitives/choice), [score](https://docs.typesafe.ai/primitives/score), [advanced](https://docs.typesafe.ai/primitives/advanced), [state](https://docs.typesafe.ai/concepts/state), [how to build with System One](https://docs.typesafe.ai/concepts/how-to-build-with-system-one), [confidence](https://docs.typesafe.ai/confidence), [patterns](https://docs.typesafe.ai/patterns), [model jaggedness jev-1.13](https://docs.typesafe.ai/model-jaggedness/jev-1.13), [Python SDK](https://docs.typesafe.ai/sdk/python), cookbooks [llm_guardrails](https://docs.typesafe.ai/cookbooks/llm_guardrails), [skill_suggestion](https://docs.typesafe.ai/cookbooks/skill_suggestion), [function_calling](https://docs.typesafe.ai/cookbooks/function_calling)
- [TypeSafe launch post](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- [Simon Willison, 21 Sep 2026](https://simonwillison.net/2026/Sep/21/jev/)
- [Wikipedia: Jev (AI model)](https://en.wikipedia.org/wiki/Jev_(AI_model))
- [DataCamp: System One models](https://www.datacamp.com/blog/system-one-models-jev), [Kingy AI review](https://kingy.ai/blog/typesafe-jev-review-the-ai-model-that-doesnt-generate-text/), [LangChain harness post](https://www.langchain.com/blog/building-a-harness-with-jev), [Pydantic AI integration](https://pydantic.dev/docs/ai/models/typesafe/)
