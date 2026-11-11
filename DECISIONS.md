# Decisions

Assumptions made where SPEC.md was unclear. The owner reviews this file at the end.

Format for new entries:

```
## <short title>
- Unclear: what SPEC.md did not say
- Chose: what was done
- Why: one line
```

## Decided before the build started

- Submit feedback: theoretical tasks show the error; algorithm tasks show "X of Y tests passed".
- Strike counter resets to 0 on restore.
- Auto-submit at timer expiry counts as an attempt.
- A fully passed task locks.
- Staying away: every further 30 s out of fullscreen or focus is another strike.
- Time bands fall linearly between best, average and worst; 40 after worst.
- No second-monitor check. No PIN for students.
