# No Prediction Policy

This file is intentionally retained for compatibility with older references, but `hltv-cs2-data` is now data-only.

The skill must not append `Model Inference`.

Never output:

- match win probability;
- map win probability;
- winner lean or favored side;
- score prediction;
- Veto prediction or hypothetical map pool;
- betting advice, odds analysis, EV, Kelly, stake sizing, max buy price, or strategy recommendation.

If the user asks for prediction or probability, return the factual data pack and decision inputs only, then add:

```text
本 skill 只输出数据层；胜率判断由调用模型或用户策略完成。
```

Completeness labels such as `complete`, `usable`, `partial`, and `blocked` may still be used to describe data quality, but they do not grant permission to output probabilities.
