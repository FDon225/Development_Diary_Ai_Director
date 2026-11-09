# Development Diary – English Summary

**Bachelor thesis:** *Entwicklung einer zentralisierten Komponente für dynamische Schwierigkeitsanpassung in Unity* (Development of a centralised component for dynamic difficulty adjustment in Unity)
**Author:** Felix Donath, Software Engineering, Hochschule Heilbronn
**Period covered:** 7 October 2026 – 9 November 2026

---

## About this document

This is a condensed English summary of the German development diary (`Entwicklungstagebuch.md`, about 330 KB, 155 dated entries). **The German diary is the authoritative record.** Where this summary and the original differ, the original applies.

The diary was kept in chronological order with date and time. It deliberately records failures, discarded approaches and untested assumptions as well as successes. Most entries were written by the AI assistant Claude (Anthropic), which supported programming, data analysis and drafting. They were written on the author's instructions and during joint work sessions. Decisions taken by the author are marked as such in the original. Entries also state openly when code had not been compiled or played at the time of writing, because the assistant had file access to the project but no direct access to the running Unity editor.

---

## 1. Project at a glance

| Item | Content |
| --- | --- |
| Goal | A working prototype of an AI Director in Unity, built as a central component decoupled from game content, plus a first test of its effect on player experience |
| Research question 1 (FF1) | How can an AI Director be implemented in Unity as a centralised component, decoupled from game content, for dynamic difficulty adjustment? |
| Research question 2 (FF2) | Does player experience differ measurably when the same prototype is played with an active AI Director instead of a fixed difficulty? |
| Test game | 2D top-down maze game with zombies, vines, healing potions, fog and an ogre boss arena; the maze is regenerated every round |
| Director design | Pull metrics through an interface, compute a tension and an overwhelm score, run a four-phase state machine (rest, build-up, peak, recovery) based on the Left 4 Dead director (Booth, 2009), send actions as events to independent receivers |
| Study | 10 participants, within-subjects, variant A (director) vs. B (fixed difficulty), 10 minutes each, counterbalanced order, SoSci Survey questionnaire |
| Submission | 18 November 2026 |

### Timeline

| Dates | Stage | Main content |
| --- | --- | --- |
| 7 Oct | Setup, stage 1 | Literature search, project structure, attempts to connect the assistant to the Unity editor, director core, greybox maze, first tuning |
| 8–9 Oct | Stage 2 | Healing potions, fog colouring, vines, ogre boss, flower guide, music, skill estimation and its first recalibration |
| 11 Oct | Stage 3 (first Windows build) | Windows build, colleague test, time limit ("corruption"), Supabase database, timeline logging, tension smoothing |
| 11–12 Oct | Stage 4 (web builds 0.5.0–0.8.0) | WebGL build, boss rage action, peak-phase rules, collision fixes, potions in tension and rating |
| 13 Oct | Stage 5 (build 1.0.0) | Boss rage made dodgeable, rating stretched, death pause |
| 14 Oct | – | Working rules recorded, database question open, questions to the supervisor |
| 15 Oct | Stage 6 (build 1.1.0, study version) | Research questions fixed, study design, variant A/B switch, first draft of the thesis |
| 20–24 Oct | Study preparation | Trial run, cosmetic fog and music cycle in variant B, SoSci Survey set up |
| 24–29 Oct | Study | 10 sessions on one PC |
| 31 Oct | Analysis | Statistical analysis, results written into chapter 5.2 |
| 5–6 Nov | Revision | Changes after supervisor feedback, several self-review passes, database exports re-checked |
| 9 Nov | Final revision | Page numbers and literature, two simulated expert reviews, proofreading, final PDF |

---

## 2. Working method and rules

- **Iteration loop:** a requirement or observation from a play test, then a code change by the assistant, then testing in the editor by the author, then a diary entry. The assistant could not see the running editor, so every change was marked "not compiled / not played" until the author confirmed it.
- **Rules set by the author (14 Oct):**
  - Answers in German.
  - Every change goes into the diary, including failures.
  - Sources need exact links.
  - Never request or accept secret database keys (only the publishable key is used in the game).
  - The author raises the build version himself and runs destructive SQL himself.
  - Ask before downloads.
  - Browsers are read-only when the assistant uses computer control.
  - Until the supervisor confirmed the research question, no A/B switch was built.
- **Known tool issue:** file transfers to the author's PC sometimes delivered an outdated file. After every transfer, file sizes were checked and the transfer repeated if needed.

---

## 3. Development stages

### Stage 1 – Core and greybox (7 Oct)

- **Tool integration failed.** Several attempts to connect the assistant directly to the Unity editor (Unity MCP server, AI Gateway, manual configuration in Claude Desktop) did not work. The integrations expected a locally running command-line instance in the project folder. Development therefore continued via shared folder access, and the author did all testing in the editor. The thesis lists this under "problems and solutions" (chapter 4.8).
- **Core of the director:** The component `AIDirector` was built with one central design decision: the director never changes game objects itself. It only sends actions as events. Metrics come through an interface (pull), actions go out as events (push).
- **Game basics:** maze generator with placeholder shapes, player and boss as separate components, a portal to a boss arena, enemy spawning, melee and ranged combat, a round loop, an orientation sensor for "lost", a first direction arrow, and fog.
- **`DirectorMemory`:** keeps the skill estimate across rounds. The author decided to keep a fixed per-round difficulty increase separate from the adaptation to the player.
- **First tuning problem:** The director stayed almost always in the build-up phase and almost never reached the peak. Fixes were a faster build-up, a time-based escalation term, a maximum peak duration and a minimum rest duration. The recovery threshold (0.65) was also unreachable with a maximum overwhelm of about 0.7, so it was lowered to 0.4.

### Stage 2 – Game content, interventions and skill estimation (8–9 Oct)

- **New content:**
  - Healing potions, dropped by the director in the recovery phase.
  - Fog colour by phase.
  - Vines breaking out of the ground in the peak phase and during the boss fight.
  - The boss became an ogre that throws rocks and summons enemies.
  - The direction arrow became a trail of flowers.
  - Music generated in code for the title screen, the phases and the boss arena.
- **Many small play-test adjustments:**
  - Zombie reach and warning times.
  - Rock spread angle.
  - Potion rarity.
  - Health carried into the next round.
  - No spawns during the rest phase.
- **Skill estimation stuck:** Analysis of 105 logged rounds showed a mean of 0.64 with a standard deviation of only 0.09. A bug also counted one round twice. The assumed cause was a feedback loop: a good player gets more pressure, overwhelm rises, the rating drops and the estimate settles. The rating was re-weighted towards damage and kills, and the estimate is now updated continuously during the round, with limits against oscillation.
- **Note recorded on 8 Oct:** the original concept paper is not binding; the research question is. The research question stayed open until the supervisor replied.

### Stage 3 – First Windows build and preparation for external tests (11 Oct)

- **Build bug:** After a round change, music and fog were missing in the build. The cause was that `Destroy` only takes effect at the end of the frame, so the newly built director deleted itself as a duplicate. The fix was `DestroyImmediate`.
- **Author test:** a 14-round run without errors in the log. Health and damage source are logged since then.
- **Colleague test:** three runs, including one with a macro that killed 1154 enemies in 54 minutes. The director stayed stable, but such rounds would distort the skill estimate. Result: a time limit in the maze ("corruption") after 120–180 s, whose damage does not count for the measurement.
- **Infrastructure:**
  - Consent text in the game.
  - Upload of round data to a Supabase database. Insert-only access, no personal data.
  - Timeline log every 5 seconds (`DirectorTimeline.csv`).
- **Changes decided by the author:**
  - Tension is now smoothed (time constant 1.5 s), after he observed sudden jumps.
  - Overwhelm weights now depend on the skill estimate.
  - "Lost" no longer counts for the skill estimate, because it depends on the random maze.

### Stage 4 – Web builds 0.5.0 to 0.8.0 (11–12 Oct)

- **WebGL build:** lets testers play in the browser without downloading anything.
  - The first web build produced no files.
  - Later the portal did not trigger, because code stripping had removed collider classes. Stripping was disabled for WebGL.
- **New director action "boss rage":** the ogre becomes faster for 6–8 s when the player is doing well.
- **Further changes:**
  - Enemies summoned in the arena no longer count as kills or as enemy pressure, because players could farm them.
  - Peak phase: the escalation term is carried into it, it ends only at clearly higher overwhelm, and the first two hits do not count.
  - Wall colliders were merged into strips and moved to foot height, and a corner assist was added, because the character got stuck on corners.
- **Potion reserve (author's decision, 12 Oct):** the potion stock now raises the tension score and gives a bonus in the rating. It is added on top of the existing weights, so the earlier calibration stays valid for players without potions.
- **Database:** about 95 web rounds from ten participant IDs in builds 0.6.0–0.8.0 became the main calibration data.

### Stage 5 – Build 1.0.0 (13 Oct)

- **Boss rage too hard:** In the 95 web rounds, rounds with rage ended in death 44 % of the time, rounds without rage only 7 %. 18 of 22 deaths happened during a rage phase, and testers said they could hardly dodge.
  - Longer warning.
  - The ogre stays slower than the player.
  - Shorter hit reach.
- **Rating stretched:** The skill estimate in the web rounds sat at about 0.6 with a standard deviation of 0.08 and barely separated strong from weak players. The rating is now stretched around 0.55. Skill values before and after build 1.0.0 are therefore not directly comparable.
- **Death pause:** 3 s pause and a death sound before the in-game questionnaire, because testers struggled with the instant pop-up.
- **Beta data:** build 1.0.0 brought 23 more rounds from seven participant IDs.

### Stage 6 – Build 1.1.0, study version (15–24 Oct)

- **Research questions and study design (author, 15 Oct):**
  - One PC.
  - Ten invited participants.
  - Each plays once with and once without the director, 10 minutes per variant.
  - Questionnaire in SoSci Survey.
- **Implementation:**
  - Variant switch A/B.
  - "New participant" button. Previously the participant ID was generated once per computer, which would have given all ten participants the same ID.
  - 10-minute session clock and an end-of-session code to link game data with the questionnaire.
  - Debug keys only in development builds.
- **Variant B:** the director keeps measuring but changes no phase, sends no actions and triggers no boss rage. Its fixed difficulty was derived from the author's pilot timeline data (1294 rows): 2.65 enemies in round 1 and about +0.65 per further round, vines every 40 s and potions every 150 s.
- **Blinding (24 Oct):** In a first test, variant B was recognisable by its constant fog. Following supervisor feedback, B now runs a purely cosmetic fog and music cycle on a randomised schedule, with phase lengths taken from the pilot data. It has no effect on enemies, loot or difficulty.
- **Build number:** The study build kept the internal version number 1.0.0, because the number in the Unity project settings was not raised. Study sessions can still be identified by platform, condition and session columns.

---

## 4. Selected problems and solutions

| Stage | Problem | Cause | Solution |
| --- | --- | --- | --- |
| 1 | No live connection between assistant and Unity editor | Plugins expect a local command-line instance in the project folder | Work via file access; testing by the author |
| 1 | Project no longer compiled (354 errors) | A plugin's dependencies were deleted before the plugin itself | Plugin removed properly via the Package Manager |
| all | Outdated file versions after transfer | Unknown | File sizes checked after every transfer |
| 1 | Ambiguous type names | Class and namespace both called `AIDirector` | Fully qualified names with `global::` |
| 1 | Director never entered recovery | Threshold 0.65 vs. maximum of about 0.7 | Threshold lowered to 0.4 |
| 2 | Skill estimate stuck around 0.48 | Formula saturation and assumed feedback loop | Per-minute values, new weights, live updating |
| 3 | No music and fog after round change in the build | `Destroy` acts at end of frame; new director deleted itself | `DestroyImmediate` when rebuilding |
| 3 | Macro test farmed 1154 enemies in 54 minutes | No time limit in the maze | "Corruption" after 120–180 s |
| 4 | Portal did not work in the web build | Code stripping removed collider classes | Stripping disabled for WebGL |
| 4 | Character stuck on free corners | Collider in sprite centre, seams between wall blocks | Merged wall colliders, foot-level collider, corner assist |
| 4 | Boss rage active in 82 % of death rounds | Raging ogre as fast as the player | Slower rage, longer warning (build 1.0.0) |

---

## 5. Study (24–29 October) and analysis (31 October)

- **Data:**
  - 10 participants aged 28–38.
  - 20 game sessions and 40 questionnaires.
  - A trial run by the author on 20 October was excluded.
  - Game data come from the local log files of the study PC. For the first participant the upload to the database failed, but the local files are complete.
  - One variant-B session contains only about 440 instead of about 590 seconds of logged play time. The cause is unclear, and the session was kept, with a robustness check excluding it.
- **Data cleaning:** Two participant IDs were mistyped in the questionnaire, and some session digits in the code were wrong. Questionnaires were therefore matched to sessions by ID, variant letter and time; all 20 matched uniquely.
- **Analysis script:** `auswertung_ff2.py` (Python, pandas, SciPy). It was written at the author's request only after the data were collected; the author did not want a pre-built script.
  - Paired Wilcoxon signed-rank tests with effect size r = Z/√n.
  - Holm correction for six main items. These six items were chosen when the script was written, after data collection; there was no preregistration.
  - Pitman-Morgan test for differences in spread between participants.
  - A later rank-based spread check, robustness check, and an exact permutation test, which confirmed all p-values.
- **Main findings:**
  - **Adaptation:** the director adapted the number of enemies to its skill estimate (Spearman ρ = 0.70).
  - **Spread:** variance between participants was smaller with the director. On closer inspection this rests mainly on one participant (damage per minute 0.95 without vs. 0.43 with the director) and, for minimum health, on a floor effect.
  - **Ogre fight:** rated as more appropriately difficult with the director (p = 0.031). This item was not among the six main items, and the effect disappears without the incomplete session, so it is treated as an indication only.
  - **No differences** in fun, immersion and boredom; better fit and less frustration only as non-significant trends.
  - **Preference:** 7 of 10 preferred variant A.
- **Limitations recorded:**
  - Variant A also differs in content (flower guide, boss rage, director-triggered potions), and six participants mentioned the flowers.
  - Only single-blind: participants were blinded, the experimenter was not.
  - The skill estimate is not validated and restarts at 0.5 after each death.
  - Small, invited sample.
  - The screen and audio recordings had no written consent. They are not analysed or quoted in the thesis and will be deleted after the examination procedure.

---

## 6. Writing and revision (31 October – 9 November)

- **31 Oct:** Results were written into chapter 5.2; screenshots and the author's concept sketches were added as figures.
- **5 Nov – supervisor feedback (Prof. Dr. Tim Reichert):**
  - No development time spans in the thesis.
  - Everything backed by sources.
  - More DDA literature and more AI-Director theory.
  - Clearly separate the Left 4 Dead phase model from the author's own design decisions.
  - Version history as its own subsection.
  - No first person.
  - A list of tables, with every table referenced in the text.
  - Own figures labelled as such.
  - All implemented: new section on directors as central control instances (Riedl & Bulitko, Alien: Isolation, RimWorld, MAPE-K), a comparison table of Booth's phases vs. the prototype's phases, and the development history as chapter 4.7. Versions below 1.0 are now called alpha, from 1.0 beta.
- **5–6 Nov – self-review passes:**
  - Robustness check without the incomplete session.
  - The spread finding was weakened after a rank-based check.
  - Skill validity and the differences between variants A and B added to the limitations.
  - Statistical method sources added.
  - All earlier database exports re-checked (round counts per build corrected; the first participant's data were never in the database).
- **9 Nov – literature:**
  - Page numbers checked against full texts where available.
  - One textbook (Döring & Bortz) was only licensed via another university and was replaced by an open-access source (Lakens, 2013).
  - A source without accessible full text (Paraschos & Koulouriotis) was removed.
  - New sources on AI Directors were read and integrated: Magerko et al. 2004, Bulitko et al. 2015, Yu et al. 2022 and 2024, Bladh & Usakauskas 2025, Dong 2025, Champandard 2009, Denisova & Cairns 2015 and Tomczak & Tomczak 2014.
- **9 Nov – two simulated expert reviews:** Main changes after them:
  - The pre-specification of the main items was corrected to "fixed after data collection".
  - Statistical details were documented.
  - Recording quotes were removed.
  - The appendix was restructured (A–G).
  - The tension formula and rating weights were explained from code and diary.
  - Extensibility was rated "not met", and static memory and missing automated tests were discussed.
  - A new timeline figure from a real study session was added.
  - The abbreviation list was completed, and the text was proofread.

---

## 7. Data and files

| File | Content |
| --- | --- |
| `Entwicklungstagebuch.md` | Full German development diary (authoritative) |
| `Development_Diary_Summary_EN.md` | This summary |
| Source code, build 1.1.0 | Unity C# scripts of the study version |
| `auswertung_ff2.py` and output tables | Statistical analysis of the study (FF2) |
| `DirectorRounds.csv`, `DirectorTimeline.csv` | Game logs of the study (random participant IDs only) |
| SoSci Survey export | Questionnaire data (random participant IDs only) |
| `Supabase_Komplett_Studie_Build_1.1.0.sql` | Database schema and access rules |

Screen and audio recordings are not part of the repository.
