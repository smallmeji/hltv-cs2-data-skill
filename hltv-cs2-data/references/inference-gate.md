# Data-Pack Boundary

This file is intentionally retained for compatibility with older references. `hltv-cs2-data` defines only the data-pack step.

The skill data pack must not append `Model Inference`.

Never put these inside factual data sections, `decision_inputs`, or JSON facts:

- match win probability;
- map win probability;
- winner lean or favored side;
- score prediction;
- Veto prediction or hypothetical map pool;
- downstream recommendations or conclusions.

If the user asks for prediction or probability, this skill still requires the factual data pack and decision inputs first. Anything after the data pack is outside this skill.

Completeness labels such as `complete`, `usable`, `partial`, and `blocked` may still be used to describe data quality, but they do not grant permission to output probabilities.
