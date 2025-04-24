# SOP: 使用 Helm Secrets 與 SOPS (GPG 後端) 管理 Helm Chart 敏感資料

## 1. 文件目的 (Purpose)

本標準作業程序（SOP）旨在規範使用 `helm secrets` Helm 插件及 Mozilla SOPS 加密工具（並以 GnuPG/GPG 作為其加密後端），來安全地管理 Helm Chart 中的敏感組態值（如密碼、API 金鑰）。目標是實現將加密後的敏感資料安全地存儲於版本控制系統（Git）中，並在 Helm 部署時自動解密。

## 2. 適用範圍 (Scope)

本 SOP 涵蓋以下範圍：
* 安裝與設定 `helm secrets` 插件、SOPS 工具及 GnuPG (GPG)。
* GPG 金鑰的生成、管理、匯出與匯入。
* 使用 SOPS 設定 GPG 加密規則 (`.sops.yaml`)。
* 加密、編輯、查看及版本控制包含敏感值的 Helm Values 文件。
* 使用 `helm secrets` 部署包含 GPG 加密值的 Helm Chart。
* 與 GPG 金鑰管理相關的流程與考量。

## 3. 先決條件 (Prerequisites)

執行本 SOP 前，請確保已完成以下準備工作：

* **3.1 Helm 安裝:**
    * 已安裝 Helm v3 或更高版本。 ([https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/))
    * 驗證：`helm version`
* **3.2 `helm secrets` 插件安裝:**
    * 執行命令：`helm plugin install https://github.com/jkroepke/helm-secrets`
    * 驗證：`helm secrets --version`
* **3.3 SOPS 安裝:**
    * 已安裝 SOPS 工具。 ([https://github.com/mozilla/sops](https://github.com/mozilla/sops))
    * macOS (使用 Homebrew): `brew install sops`
    * 其他系統請參考 SOPS 官方文件。
    * 驗證：`sops --version`
* **3.4 GnuPG (GPG) 安裝:**
    * 已安裝 GnuPG 工具鏈。
        * Debian/Ubuntu: `sudo apt update && sudo apt install gnupg gpg-agent`
        * CentOS/Fedora: `sudo yum install gnupg2` 或 `sudo dnf install gnupg2`
        * macOS (使用 Homebrew): `brew install gnupg`
    * 驗證：`gpg --version`
* **3.5 GPG 金鑰對準備:**
    * **每個需要解密權限的使用者和 CI/CD 系統** 都需要擁有自己的 GPG 金鑰對。
    * 如果你還沒有 GPG 金鑰，請生成一個（見步驟 4.1）。
    * 需要收集所有相關人員/系統的 GPG 公鑰指紋 (Fingerprint)。

## 4. 標準作業程序 (Procedure)

**4.1 生成/識別 GPG 金鑰與指紋**

* **檢查現有金鑰:** 列出你的私鑰和對應的指紋。
    ```bash
    gpg --list-secret-keys --keyid-format LONG
    ```
    輸出會類似：
    ```
    sec   rsa4096/XXXXXXXXXXXXXXXX # 這是 Key ID (短)
          YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY # <--- 這是你的 GPG 指紋 (Fingerprint)
    uid                 [ultimate] Your Name <your.email@example.com>
    ssb   rsa4096/ZZZZZZZZZZZZZZZZ
    ```
    記下那個長長的 `YYYY...` 字串，這就是你的指紋。

* **生成新金鑰 (如果沒有):**
    ```bash
    gpg --full-generate-key
    ```
    按照提示操作：
    * 選擇金鑰類型 (通常選 RSA and RSA)。
    * 選擇金鑰大小 (建議 4096 位元)。
    * 設定過期時間 (例如 1y 或 0 表示永不過期)。
    * 輸入你的姓名 (Real name) 和電子郵件 (Email address)。
    * **設定一個強密碼 (Passphrase)!** 這是保護你私鑰的關鍵。
    * 系統會要求你進行一些隨機操作（移動滑鼠、敲鍵盤）以生成足夠的熵。
    * 生成後，再次使用 `gpg --list-secret-keys` 確認並取得指紋。

**4.2 配置 SOPS 使用 GPG (`.sops.yaml`)**

在你的 Helm Chart 或 Git 倉庫的 **根目錄** 下創建一個名為 `.sops.yaml` 的文件。

* **目的:** 告知 SOPS 對哪些文件使用 GPG 加密，以及使用哪些人的 GPG 公鑰（通過指紋識別）。
* **創建 `.sops.yaml` 文件:** 如果有多個pgp 然後後面需註解 [可參考](https://github.com/getsops/sops#:~:text=Alternatively%2C%20you%20can%20configure%20the%20Shamir%20threshold%20for%20each%20creation%20rule%20in%20the%20.sops.yaml%20config%20with)
    ```yaml
    # .sops.yaml
    creation_rules:
      - path_regex: .*/secrets/.*\.yaml$ # 範例：只加密 secrets/ 目錄下的 yaml 文件
        # 或者 path_regex: secrets.yaml$  # 範例：只加密名為 secrets.yaml 的文件
        # encrypted_regex: ^(data|stringData)$ # 可選：進一步限制只加密 Kubernetes Secret 中的 data/stringData
        pgp: >- # 使用 >- 可以讓你在多行列出指紋，更清晰
            FINGERPRINT_ALICE,
            FINGERPRINT_BOB,
            FINGERPRINT_TEAMMATE,
            FINGERPRINT_CICD_SYSTEM
            # 將上面的 FINGERPRINT_* 替換成所有需要解密權限的人/系統的 GPG 指紋
            # 用逗號分隔，可以放在同一行或多行
    ```
* 可測試sops檔案是否正確 (有```secrets.yaml```)
    ```
    sops -e secrets.yaml
    ```
* **重要:**
    * 確保 `path_regex` 正確匹配你存放敏感資訊的 YAML 文件。
    * `pgp` 字段必須包含**所有**需要能夠解密這些文件的人員和 CI/CD 系統的 GPG 指紋。

**4.3 創建敏感值文件** (同前)

將敏感資料放入一個獨立的 YAML 文件，確保其路徑符合 `.sops.yaml` 中的 `path_regex`。

* **示例:** 創建 `secrets.yaml`
    ```yaml
    # secrets.yaml
    database:
      password: "ThisWillBeEncryptedByGPG!"
    apiCredentials:
      clientSecret: "AnotherSuperSecretTokenForGPG"
    ```

**4.4 使用 SOPS (GPG) 加密敏感值文件**

使用 `helm secrets encrypt` 命令。`helm secrets` 會調用 SOPS，SOPS 讀取 `.sops.yaml`，發現需要使用 GPG，然後使用 `pgp` 字段中列出的所有公鑰來加密文件。

* **執行加密:**
    ```bash
    helm secrets encrypt secrets.yaml
    ```
* **GPG Agent 提示:** 如果你的 GPG 私鑰有密碼保護，GPG Agent 可能會彈出視窗或在終端提示你輸入密碼以允許 SOPS 使用私鑰進行操作（即使加密也可能需要簽名）。
* **驗證:**
    * 加密成功後，`secrets.yaml` 文件內容會變成包含 SOPS GPG 元數據和密文的結構。
    * `cat secrets.yaml` 看到的 `ENC[...]` 內容現在是 GPG 加密的結果。

**4.5 將加密文件提交到版本控制 (Git)** (同前)

* **添加加密文件和 SOPS 配置:**
    ```bash
    git add secrets.yaml
    git add .sops.yaml
    ```
* **配置 `.gitignore` (重要):** 確保明文版本不會被提交。
    ```gitignore
    # .gitignore
    *.decrypted.yaml
    secrets*.plaintext.yaml
    # 或更具體地忽略你的明文檔案模式
    ```

**4.6 在 Helm Chart 中引用加密值** (同前)

在 Chart 的模板中直接使用 `.Values` 引用 `secrets.yaml` 中的路徑即可。

* **模板示例 (`templates/secret.yaml`):**
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: {{ include "mychart.fullname" . }}-api-creds
    type: Opaque
    stringData:
      # 引用來自 secrets.yaml 的值
      clientSecret: {{ .Values.apiCredentials.clientSecret }}
    ```

**4.7 使用 `helm secrets` 部署或管理 Helm Release** (同前)

**務必** 使用 `helm secrets <command>`。

* **安裝/升級/模板化:**
    ```bash
    helm secrets install my-release ./my-chart -f values.yaml -f secrets.yaml
    helm secrets upgrade my-release ./my-chart -f values.yaml -f secrets.yaml
    helm secrets template my-release ./my-chart -f values.yaml -f secrets.yaml
    ```
* **GPG Agent 提示:** 在執行這些命令時，如果需要解密（如 `install`, `upgrade`, `template`, `get values`），`helm secrets` -> SOPS -> GPG 會嘗試使用你的私鑰解密。如果私鑰有密碼，GPG Agent 可能會要求你輸入。

**4.8 編輯已加密的 Secrets 文件** (同前)

使用 `helm secrets edit` 命令。

* **執行編輯:**
    ```bash
    helm secrets edit secrets.yaml
    ```
* **流程:** 解密 -> 編輯 -> 重新加密。GPG Agent 可能會在解密和重新加密（簽名）時要求輸入密碼。

**4.9 查看已加密 Secrets 文件的明文內容** (同前)

使用 `helm secrets view` 命令。

* **執行查看:**
    ```bash
    helm secrets view secrets.yaml
    ```
* **GPG Agent 提示:** 可能會要求輸入 GPG 私鑰密碼以進行解密。

## 5. GPG 金鑰管理程序 (關鍵!)

這是確保 GPG 後端方案成功的核心。

* **查找自己的指紋:**
    ```bash
    gpg --list-keys --keyid-format LONG your.email@example.com
    ```
* **匯出你的公鑰 (分享給他人):**
    ```bash
    # 將 <你的郵件或指紋> 替換成你的實際值
    gpg --export --armor <你的郵件或指紋> > my-public-key.asc
    ```
    將生成的 `my-public-key.asc` 文件安全地發送給需要用你的公鑰加密的人（例如，添加到 `.sops.yaml` 的團隊成員）。

* **匯入他人的公鑰:**
    ```bash
    # 假設你收到了名為 teammate-pubkey.asc 的公鑰文件
    gpg --import teammate-pubkey.asc
    ```
    匯入後，你才能使用這個公鑰加密文件（SOPS 會自動查找已匯入的公鑰）。

* **信任金鑰 (可選但推薦):** 為了避免 GPG 警告，你可以編輯金鑰信任級別。
    ```bash
    gpg --edit-key <要信任的指紋>
    # 在 gpg> 提示符下輸入 'trust'
    # 選擇信任級別 (例如 5 = Ultimately trust)
    # 輸入 'save' 退出
    ```

* **保護你的私鑰:**
    * **強密碼:** 為你的私鑰設定一個難以猜測的密碼。
    * **權限:** 確保你的 `~/.gnupg` 目錄權限安全 (通常是 `drwx------`)。
    * **備份:** 安全地備份你的私鑰和撤銷證書。

* **共享指紋:** 通過可信賴的方式（例如，面對面、加密通訊）告知團隊成員你的 GPG 指紋，以便他們將其添加到 `.sops.yaml` 文件中。

* **CI/CD 系統的 GPG 金鑰:**
    1.  為 CI/CD 系統生成一個專用的 GPG 金鑰對。
    2.  將其**公鑰指紋**添加到 `.sops.yaml` 中。
    3.  將其**私鑰**（通常使用 `gpg --export-secret-keys --armor <CI_KEY_FINGERPRINT>` 匯出）和**密碼**安全地存儲在 CI/CD 系統的 Secrets Management 功能中（例如 GitHub Secrets, GitLab CI/CD Variables, Jenkins Credentials）。
    4.  在 CI/CD Pipeline 執行時，從 Secrets Manager 中讀取私鑰和密碼，使用 `gpg --import` 將私鑰導入到臨時的 GPG 環境中，並可能需要配置 GPG Agent 或直接提供密碼給 SOPS/GPG。

## 6. 故障排除 (Troubleshooting GPG Specific Issues)

* **`ERROR secrey key not available` / `gpg: decryption failed: No secret key`:**
    * **原因:** 你的本地 GPG Keyring 中沒有用於解密該文件所需的私鑰，或者 GPG Agent 無法訪問該私鑰。
    * **解決:** 確認 `.sops.yaml` 加密時使用的指紋對應的私鑰在你本地存在 (`gpg --list-secret-keys`)。確認 GPG Agent 正在運行且配置正確。
* **`gpg failed to sign the data` / 不斷提示輸入密碼:**
    * **原因:** GPG 操作（如簽名或解密）需要私鑰密碼，但無法獲取或輸入錯誤。
    * **解決:** 確保輸入正確的密碼。檢查 GPG Agent 配置是否允許緩存密碼或是否能正確彈出提示。
* **SOPS 報錯 `no valid OpenPGP data found`:**
    * **原因:** 文件可能未被正確加密，或者文件已損壞。
    * **解決:** 嘗試重新加密原始明文文件。檢查文件是否被意外修改過。
* **GPG Agent 問題:** GPG Agent 負責管理私鑰密碼，如果它沒有運行或配置錯誤，會導致 SOPS 無法順利使用 GPG。查閱 GPG Agent 相關文檔進行調試。

## 7. 參考資料 (References)

* **Helm Secrets 文檔:** [https://github.com/jkroepke/helm-secrets](https://github.com/jkroepke/helm-secrets)
* **Mozilla SOPS 文檔:** [https://github.com/mozilla/sops](https://github.com/mozilla/sops)
* **GnuPG 文檔:** [https://gnupg.org/documentation/](https://gnupg.org/documentation/)
* **GPG Cheatsheet (常用指令):** [https://devhints.io/gpg](https://devhints.io/gpg)

## 8. 文件控制 (Document Control)

* **版本:** 1.0
* **制定日期:** 2025-04-24
* **制定者:** AI Assistant
* **審核者:** (待審核)