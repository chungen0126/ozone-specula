# ozone-specula

用 GitHub Actions 對 Ozone 跑 [Specula](https://github.com/specula-org/Specula)
（AI 產生 TLA+ spec、TLC model checking、trace validation、bug 確認）。
做法跟 [HDDS-15926](https://issues.apache.org/jira/browse/HDDS-15926) 的子任務相同，
只是改在自己的 GitHub 上跑，預設對 `chungen0126/ozone` 的 `master`。

Workflow：[`.github/workflows/specula.yml`](.github/workflows/specula.yml)，只能手動觸發（`workflow_dispatch`）。

## 一次性設定

1. 設定 Claude 的認證，二選一，存成 repository secret：
   - API key（依 token 計費，單一目標約 $50–90）：

     ```bash
     gh secret set ANTHROPIC_API_KEY -R chungen0126/ozone-specula
     ```

   - Claude 訂閱：先在本機跑 `claude setup-token` 取得 token，再

     ```bash
     gh secret set CLAUDE_CODE_OAUTH_TOKEN -R chungen0126/ozone-specula
     ```

2. 決定 repo 要不要設成 private。public repo 的 Actions log、job summary
   和 artifact 任何人都看得到，裡面會有還沒公開的 bug 細節。
   若結果可能是安全漏洞，依 Ozone 的 `SECURITY.md` 不該公開。
   private repo 的免費 runner 規格較小，分鐘數也有額度，見下一節。

## Runner 資源

| Runner | CPU / RAM | 單一 job 上限 |
|---|---|---|
| `ubuntu-latest`（public repo） | 4 核 / 16 GB | 6 小時 |
| self-hosted | 看你的機器 | workflow 設 72 小時 |

Specula 建議至少 32 GB RAM；HDDS-15926 的 run 用 `--tlc-memory-limit=28G`，
單一目標要跑 1.5–5.5 小時。所以在 `ubuntu-latest` 上預設值收斂成：
TLC 10G、3 個 worker、一次 1 個 agent、Specula 跑 320 分鐘後停下來上傳結果。
範圍窄的目標可以跑完；範圍大的目標建議用 self-hosted runner：
在 repo 的 Settings → Actions → Runners 註冊一台 Linux（或裝了 GNU coreutils 的 macOS）機器，
觸發時把 `runner` 設成它的 label，再調高 `tlc_memory_limit`、`tlc_worker_limit`
和 `time_limit_minutes`。

## 跑一個目標

先把 guidance 檔放進 [`guidance/`](guidance/README.md)，push 之後：

```bash
gh workflow run specula.yml -R chungen0126/ozone-specula \
  -f target=om-key-lifecycle \
  -f guidance=guidance/om-key-lifecycle.md \
  -f ozone_ref=master
```

常用參數：

| 參數 | 預設 | 說明 |
|---|---|---|
| `ozone_repo` / `ozone_ref` | `chungen0126/ozone` / `master` | 要驗的 Ozone 程式碼，例如 `apache/ozone` / `HDDS-11233` |
| `runner` | `ubuntu-latest` | self-hosted 時填 label |
| `model` / `effort` | `claude-opus-5-5[1m]` / `medium` | 帳號沒有 1M context 時改成 `claude-opus-5-5` |
| `reviews_and_debate` | `true` | `--enable-reviews --confirm-debate`，跟 HDDS-15926 一致 |
| `specula_ref` | 固定的 Specula commit | 要更新 Specula 時改這裡 |

看進度：

```bash
gh run watch -R chungen0126/ozone-specula
```

## 結果

每次 run 會上傳一個 artifact `specula-<target>-<run id>-<attempt>`（保留 30 天）：

```bash
gh run download <run id> -R chungen0126/ozone-specula
```

- `pipeline-summary.md`：整個 run 的狀態，也會出現在 job summary
- `<target>/.specula-output/summary.md`：這個目標的結果
- `<target>/.specula-output/confirmed-bugs.md`、`bug-severity.md`：確認過的 bug
- `<target>/changes.patch`：agent 對 Ozone 加的 harness 和重現測試

Specula 的 finding 是假設，要自己對照程式碼和重現測試確認之後，才開 Ozone 的 Jira。
