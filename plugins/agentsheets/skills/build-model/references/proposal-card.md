# Proposal card contract

Inspect enough current state to make the proposal specific. Do not create or change the artifact
before approval. Submit one complete revision when asked. Never say a proposed artifact exists.

Call `propose_plan` with a compact card:

- `title`: action-led, sentence case, and specific.
- `message`: a short present-tense status within the tool schema's 80-character limit.
- `summary`: an optional one-line explanation in plain product language.
- `deliverables`: each output's display `name`, supported `type`, one-line `description`, and actual
  sources. Set `connectionType` only for a known connected integration; omit it for derived model
  inputs. A model variable, formula, or table remains derived even when its data came from an
  integration. If the direct source is uncertain, omit optional `sources` rather than guessing.
- `assumptions`: only choices that could change the work. Include source limits that would make an
  output a proxy or estimate; never weaken it silently. Give each a label, a recommended `choice`,
  and two to four different `options`. Do not repeat `choice` in `options` or add “recommended”;
  selection already shows the default. Use `blocking: true` only when no safe default exists. Keep
  five assumptions or fewer. If a blocking choice has no safe business answer, omit `choice`, list
  the available `options`, and do not invent a value.

Omit absent optional fields instead of serializing `null`. Keep summaries and deliverable
descriptions to one sentence without trailing periods. Lead with what the user gets, not the
agent's reasoning.
