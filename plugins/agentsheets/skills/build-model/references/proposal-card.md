# Proposal card contract

Inspect enough current state to ground the proposal, but do not create or modify the artifact until
the user approves. Submit one complete revision when asked, and never claim a proposed artifact exists.

Call `propose_plan` with a compact card:

- `title`: substantive, action-led, sentence case, and specific about the output.
- `message`: a short present-tense status within the tool schema's 80-character limit.
- `summary`: an optional one-line explanation in plain product language.
- `deliverables`: each output's display `name`, supported `type`, one-line `description`, and actual
  sources. Set `connectionType` only for a known connected integration; omit it for derived model
  inputs. A model variable, formula, or table remains derived even when its data came from an
  integration. If the direct source is uncertain, omit optional `sources` rather than guessing.
- `assumptions`: only choices that could change the work, including source limits that would turn
  the requested output into a proxy or approximation. Put that choice on the card; never quietly
  weaken the deliverable. Give each a label, a recommended default in `choice`, and two to four
  distinct alternatives in `options`. Do not repeat `choice` in `options` or add labels such as
  `(recommended)`; the selected state already shows the default. Use `blocking: true` only when no
  safe default exists, and aim for five assumptions or fewer. For a blocking assumption without a
  safe business answer, omit `choice` and put the available answers in `options`; never invent a
  business value.

Omit absent optional fields instead of serializing `null`. Keep summaries and deliverable
descriptions to one sentence without trailing periods. Lead with what the user gets, not the
agent's reasoning.
