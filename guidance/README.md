# Guidance 檔

每個 Specula 目標一個 guidance 檔，限定要建模的模組、程式進入點和要檢查的性質。
沒有 guidance 時 Specula 會自己決定範圍，對 Ozone 這麼大的 codebase 會跑不完。

取得方式：

- HDDS-15926 子任務附的 `<target>.guidance.md`，例如
  [HDDS-17428](https://issues.apache.org/jira/browse/HDDS-17428) 的 `dn-short-circuit-read.guidance.md`。
  下載後放到這個目錄，觸發 workflow 時 `target` 用同一個名字。
- 自己寫：從 Specula 的
  [template](https://github.com/specula-org/Specula/blob/main/docs/modeling-guidance-template.md)
  和 [example](https://github.com/specula-org/Specula/blob/main/docs/modeling-guidance-example.md) 開始。

guidance 裡的行號是寫的當下的 commit，換 commit 會偏移；Specula 會重新分析原始碼，不依賴這些行號。
