# 03-Stack — SR Tracker

| Problem                                                                     | ID  | Difficulty | Box | Last Reviewed | Next Due   | Reps | Lapses |
| --------------------------------------------------------------------------- | --- | ---------- | --- | ------------- | ---------- | ---- | ------ |
| [[0020-Valid-Parentheses\|Valid Parentheses]]                               | 20  | easy       | 0   | -             | -          | 0    | 0      |
| [[0150-Evaluate-Reverse-Polish-Notation\|Evaluate Reverse Polish Notation]] | 150 | medium     | 0   | -             | -          | 0    | 0      |
| [[0155-Min-Stack\|Min Stack]]                                               | 155 | medium     | 0   | -             | -          | 0    | 0      |
| [[0739-Daily-Temperatures\|Daily Temperatures]]                             | 739 | medium     | 2   | 2026-08-24    | 2026-08-27 | 2    | 1      |
| [[0853-Car-Fleet\|Car Fleet]]                                               | 853 | medium     | 0   | -             | -          | 0    | 0      |

### Notes

**Daily Temperatures** — Confusion: 無法手動模擬單調 stack 的 push／pop 過程，看到 `<=` 版程式碼無法推出 `[30,30,41]` 的輸出。Key point: stack 存「還在等更暖天」的 index；新的一天 `i` 進場就把棧頂所有不夠暖的日子彈出結算 `ans[j] = i - j`。彈出條件必須是**嚴格 `<`**——等溫不算更暖，同溫日要留在棧裡繼續等。寫成 `<=` 會讓同溫日互相提早結算，答案偏小（`[30,30,41]`：`<=` 得 `[1,1,0]`，正解 `[2,1,0]`）。下次改用不同輸入 + 要求口述逐格 trace 來測。
