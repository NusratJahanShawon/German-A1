# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

"Deutsch üben" is a German vocabulary trainer (nouns + articles, A1 words, verbs) with English and Bangla meanings. The entire app is one self-contained file, `german-nouns-quiz.html`: inline CSS, inline vanilla JS, and all word data. No build step, package manager, tests, or linter. It's hosted on GitHub Pages from the `main` branch (`index.html` redirects to the app).

To run it, open the HTML file in a browser. The only external resource is Google Fonts (Bricolage Grotesque, plus Noto Sans Bengali for Bangla text). Pronunciation uses the browser's `speechSynthesis` API. `voiceScore` ranks the device's voices so natural ones ("Natural", "Online", "Premium", "Enhanced", Google) win over robotic or novelty voices. Voices don't report gender, so `vGenderOf` guesses it from known names and Android voice codes (`MALE_V` / `FEMALE_V`), and male voices are preferred (the learner asked for a male voice). `voiceChoices` lists every voice plus a lower-pitched "deeper" version of each non-male one, used by default when the device has no male voice; the chosen pitch is in `vPitch`. The German and English dropdowns live inside every `.voiceHint`, and the choice is saved in `localStorage` (`voicePick:de`, `voicePick:en`).

## File layout (top to bottom)

1. `<style>`: color tokens on `:root`, with dark mode under both `prefers-color-scheme` and `[data-theme="dark"]`. Article colors are `--der` (blue), `--die` (red) and `--das` (green); verbs use `--verb` (orange). Each has a `-soft` variant. Classes like `art-der` / `a-der` color words by article.
2. Static HTML: one `<section class="panel">` per view (`setup`, `rules`, `nteach`, `list`, `vlearn`, `vteach`, `vsetup`, `vlist`, `quiz`, `results`). JS fills most of their content.
3. `<script>`, in this order:
   - **Noun data**: `CATS` (topic names, indexed by position) and `RAW`, a pipe-delimited template string with one word per line: `article|word|english|bangla|topicIndex|note|group`. The article is empty for non-nouns. `group` marks words whose meanings are too close to be distractors for each other; `SAME` does the same job keyed by English meaning. Parsed into `WORDS`.
   - **Verb data**: `VRAW`, with fields `n|infinitive|english|bangla|aux|partizip II|kind|separable prefix|changed stem|example DE|example EN|note|group`. `kind` is one of `H` (helper), `Mod` (modal), `X` (mixed), `S` (strong/irregular), `R` (regular). Parsed into `VERBS`.
   - **Shared helpers**: the `$` id lookup, the global settings object `S` (mode, lang `en`/`bn`, cats, count, auto-speak, rate, vmode, vset), `shuffle`, and `esc`. **Run all interpolated text through `esc()`**, because views are built with `innerHTML`. Settings chips are `.chips[data-key=<S key>] .chip[data-v=<value>]` and are wired by one delegated click handler.
   - **Voice**: `say(text, btn)`. Any element with a `data-t` attribute speaks that text on click.
   - **Shared quiz engine**: `runQuiz(questions, module)` → `renderQ` / `answer` / `next` / `results`. Each question is a plain object: `{key, prompt, word, gloss, layout, opts:[{v,html,cls}], ans, sayBefore, sayAfter, reveal(), feedback(ok), miss, redo()}`. Keys 1–4 pick answers (keydown handler). To add a new kind of quiz, write a builder that returns these objects; don't change the engine.
   - **Nouns module**: `renderCats` and `renderList`; `RULES`, the article-guessing rules, each with `lvl` (Always/Usually/Weak/None), `test(w)` and `extra` example words made with `ex()`; `nounQ` builds article and meaning questions, with distractors from the same topic first; `startNounQuiz`.
   - **Verbs module**: conjugation is *generated*, not stored. `conjRaw` builds present-tense forms from stem rules and handles separable prefixes, `-ern`/`-eln`, e-insertion, and s-sounds. Exceptions come from the lookup tables `FORMS` (fully irregular), `CHG` (vowel-change labels), `INSEP` (inseparable prefixes) and `MOVE`/`CHANGE`/`ALWAYS_SEIN` (explain haben vs. sein). `vQ` builds verb questions; `VSETS` filters verbs by number range; `VLESSONS` is the content of the Learn tab: each lesson's `grp` is an index into `VGROUPS` (the numbered A1 chapters), and `VCHECK` is the exam checklist at the end. The verb rule lessons have a spoken script in `LECT` (same order and count as `VLESSONS`).
   - **Teacher** (the `nteach` and `vteach` tabs): spoken lessons. `course(id)` builds one of three courses: `vrules` (from `VLESSONS` + `LECT`), `verbs` (generated per verb by `verbSteps`) and `nouns` (`NINTRO`, then one lesson per `RULES` entry, using the hand-written `NSCRIPT[tag]` plus example words from `rulePool`). A lesson is a list of steps, each a list of lines: `{…}` is German (German voice, tappable), other text uses an English voice, `"~"` pauses for the learner to repeat, and `"bn:"` lines are shown but not spoken. `openTeach` builds the tab's HTML; `playStep` chains the speech. Finished lessons are saved in `localStorage` under `teachDone:<course>`. A new noun rule needs an `NSCRIPT` entry, or the teacher just reads its `desc`.
   - **Navigation**: `MODULES` defines the top-level modules and their sub-tabs. `show(viewId)` renders the view and toggles `hidden` on panels. The URL hash (`#nouns` / `#verbs`) picks the module. `lastView` remembers each module's tab, including a quiz in progress.

## Conventions and gotchas

- Topic membership is the **index** into `CATS`. `NOUNCATS` (0–15), `WORDCATS` (16–21) and the `SET1` / `SET2` presets (the "200 nouns list" / "A1 500 words list" chips) rely on that order, so append new topics instead of reordering, and update those constants.
- When adding words, keep the field count consistent, and don't let a `|` or backtick slip into a field: the parsers split on `|` and the data lives in template literals. Notes and the Bangla column can be blank but the separators must be there.
- Article rules match on the word text (regex in `test`), so a new noun can be picked up by a rule automatically. Check that it doesn't land in the wrong rule.
- UI text is English, apart from German examples and Bangla meanings. Keep the plain, learner-friendly tone of the existing copy.
- Layout must keep working on phones (safe-area insets, `clamp()` font sizes, 760px max width).
