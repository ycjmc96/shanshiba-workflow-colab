# Windows 本機 Codex 工作流

Codex CLI 在已登入 Colab 的 Windows 帳號下執行。它在工作區準備素材、呼叫同一個 `colab.exe`、收回產物，並依照人工審查結果繼續。使用者分別以 ChatGPT 登入 Codex、以 Google 登入 Colab。雲端工作區需要獨立的 Colab 授權與工作階段狀態。HTTPS 可達只代表網路可通，跨機器存取仍需在目標機器驗證。

## 啟動 Codex 並驗證現有連線

1. 在能執行 `colab exec` 的同一個 Windows 帳號開 PowerShell。若尚未安裝 Codex，可用 `npm install -g @openai/codex` 或依官方 Windows 安裝指引操作。 [Codex CLI 官方安裝指引](https://github.com/openai/codex#installing-and-running-codex-cli)
2. 在儲存庫根目錄執行以下命令。Codex 首次啟動時選擇 **Sign in with ChatGPT**。如果 Codex 沙箱阻擋 Colab CLI 存取 Windows 鑰匙圈、工作階段資料或網路，只核准該條連線檢查命令。Google 若要求重新授權，在自己的 PowerShell 執行 `colab usage`。

   ```powershell
   Get-Command colab, codex
   python .\cloud\colab_h3.py --name video-agent preflight
   codex --sandbox workspace-write --ask-for-approval on-request
   ```

3. 在 Codex 內先交付這項任務。

   > 在目前工作區執行 `python cloud/colab_h3.py --name video-agent preflight`。只使用這個既有工作階段，回報帳號及執行環境是否驗證成功。這一輪只做連線測試。

兩個位置都應看到 `COLAB_CONNECTION_OK`。預檢先透過 `usage` 驗證帳號，再對指定工作階段執行一行 Python。若 PowerShell 可以連而 Codex 內失敗，先確認兩邊解析到同一個 CLI，並檢查沙箱核准狀態。若工作階段已逾時，先回報。預檢沒有建立 VM、安裝模型或送出素材的路徑。

## 從劇本到交付

以下階段在使用者同意開始製作後才執行。已驗證 Python 可執行的 L4 工作階段，只能視為連線已通；還要另外確認 GPU 型號、H3 部署、模型授權地區及預算。專案對 G4 的效能估算不適用於 L4。 [Colab 工作階段與模型需求](README.md)

| 階段 | Codex 執行 | 繼續條件 |
| --- | --- | --- |
| 規劃 | 從 `pipeline/template_local` 複製出一個全新的專案目錄，準備劇本、讀本、鏡表及 `film.json`。以唯一專案目錄隔離現有素材和成片。 | 使用者確認劇本、鏡表與輸出位置。 |
| 首幀及配音 | 有本機模型時執行範本的 `chars masters frames`、`voices`；已有完整 Colab 環境時，可對同一個專案分段使用 `colab_h3.py --name video-agent film`。確認這次要傳的素材清單，專案目錄只放製作需要的資料。 | 使用者檢查首幀接觸表和配音，確認可以上傳至該工作階段。 |
| 分鏡渲染 | 驗證每鏡 JSON 及素材路徑，執行 `colab_plan.py` 取得估算。純 H3 模式可用 `python cloud/colab_h3.py --name video-agent batch --existing-only <新專案的分鏡 JSON>`；完整 Colab 模式按範本執行 `film <專案目錄> h3 amb`。 | 確認工作階段已安裝所需功能，使用者已核准機型、用量與本批範圍。 |
| 審片與補拍 | 依鏡號比對實際輸出、接觸表、ASR 與構圖檢查。記下 spec、seed、結果和失敗原因。重試前先查遠端是否仍在執行，避免重複送件。把補拍存到獨立嘗試目錄，再選用通過的版本。 | 人工確認角色連戲、每鏡動作和台詞。 |
| 後製與交付 | 依範本執行 `edl music ending cut review`。若階段一直在完整 Colab 執行，就在同一台 VM 完成至後製，避免本機讀到 `/content` 路徑。確認片長、字幕、聲音和完整成片，再交付影片及審片紀錄。 | 使用者看完整支成片後確認交付。是否停止 VM 另依使用者指示。 |

每次只派一個 Codex 流程操作同一個 Colab 工作階段。用同一份專案的鏡表、階段記錄及產物恢復進度。既有 `batch` 會寫入 spec 旁的輸出目錄，完整 `film` 會同步專案的 `out/`；補拍或重新剪輯前先隔離要保留的版本。連線或遠端安裝標記檢查失敗時回報目前階段，用 `--existing-only` 避免批次命令自行開機或安裝。 [多鏡範本](../pipeline/template_local/README.md)
