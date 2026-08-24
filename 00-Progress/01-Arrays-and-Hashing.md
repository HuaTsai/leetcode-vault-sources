# 01-Arrays-and-Hashing — SR Tracker

| Problem                                                             | ID  | Difficulty | Box | Last Reviewed | Next Due   | Reps | Lapses |
| ------------------------------------------------------------------- | --- | ---------- | --- | ------------- | ---------- | ---- | ------ |
| [[0001-Two-Sum\|Two Sum]]                                           | 1   | easy       | 0   | -             | -          | 0    | 0      |
| [[0036-Valid-Sudoku\|Valid Sudoku]]                                 | 36  | medium     | 0   | -             | -          | 0    | 0      |
| [[0049-Group-Anagrams\|Group Anagrams]]                             | 49  | medium     | 0   | -             | -          | 0    | 0      |
| [[0128-Longest-Consecutive-Sequence\|Longest Consecutive Sequence]] | 128 | medium     | 1   | 2026-08-24    | 2026-08-25 | 1    | 1      |
| [[0217-Contains-Duplicate\|Contains Duplicate]]                     | 217 | easy       | 0   | -             | -          | 0    | 0      |
| [[0238-Product-of-Array-Except-Self\|Product of Array Except Self]] | 238 | medium     | 1   | 2026-08-24    | 2026-08-25 | 1    | 0      |
| [[0242-Valid-Anagram\|Valid Anagram]]                               | 242 | easy       | 0   | -             | -          | 0    | 0      |
| [[0271-Encode-and-Decode-Strings\|Encode and Decode Strings]]       | 271 | medium     | 0   | -             | -          | 0    | 0      |
| [[0347-Top-K-Frequent-Elements\|Top K Frequent Elements]]           | 347 | medium     | 0   | -             | -          | 0    | 0      |

### Notes

**Longest Consecutive Sequence** — Confusion: 拿掉 `if (s.count(n - 1)) continue;` 守衛後的最壞複雜度答不出來，需要先看到程式碼、再被提示總和才推出 O(n²)。Key point: 守衛的作用是**只從序列起點往上數**。有守衛時整條連續序列只有一個起點會展開，總步數 O(n)；沒有守衛時每個數都往上數到底，`n + (n-1) + … + 1 = n(n+1)/2` 即 O(n²)。下次改測「哪一種輸入會讓無守衛版最慢」或要求直接口述這個總和。
