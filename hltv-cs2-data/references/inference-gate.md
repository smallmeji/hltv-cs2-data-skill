# Data-Pack Boundary

This file is intentionally retained for compatibility with older references. `hltv-cs2-data` is data-pack-first and has no built-in prediction model.

The skill data pack must not append `Model Inference`.

Never put these inside factual data sections, `decision_inputs`, or JSON facts:

- match win probability;
- map win probability;
- winner lean or favored side;
- score prediction;
- Veto prediction or hypothetical map pool;
- betting advice, odds analysis, EV, Kelly, stake sizing, max buy price, or strategy recommendation.

If the user asks for prediction or probability, return the factual data pack and decision inputs first. After the data pack, the calling model may continue outside the skill with its own judgment. Use this boundary label:

```text
以下判断由调用模型基于上述数据自行完成，不是 HLTV 数据库字段，也不是 hltv-cs2-data skill 输出。
```

Completeness labels such as `complete`, `usable`, `partial`, and `blocked` may still be used to describe data quality, but they do not grant permission to output probabilities.
