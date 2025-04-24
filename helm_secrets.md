# SOP: 使用 Helm Secrets 與 SOPS 管理 Helm Chart 敏感資料

## 1. 文件目的 (Purpose)

本標準作業程序（SOP）旨在規範使用 `helm secrets` Helm 插件及 Mozilla SOPS 加密工具，安全地管理 Helm Chart 中的敏感組態值（如密碼、API 金鑰）。目標是實現將加密後的敏感資料安全地存儲於版本控制系統（Git）中，並在 Helm 部署時自動解密。

## 2. 適用範圍 (Scope)

本 SOP 涵蓋以下範圍：
* 安裝與設定 `helm secrets` 插件及 SOPS 工具。
* 使用 SOPS 設定加密規則 (`.sops.yaml`)。
* 加密、編輯、查看及版本控制包含敏感值的 Helm Values 文件。
* 使用 `helm secrets` 部署包含加密值的 Helm Chart。
* 與 SOPS 相關的金鑰管理考量。

## 3. 先決條件 (Prerequisites)

執行本 SOP 前，請確保已完成以下準備工作：

* **3.1 Helm 安裝:**
    * 已安裝 Helm v3 或更高版本。 ([https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/))
    * 驗證：`helm version`
* **3.2 `helm secrets` 插件安裝:**
    * 執行命令：`helm plugin install https://github.com/jkroepke/helm-secrets`
    * 驗證：`helm secrets --version`
* **3.3 SOPS 安裝:**
    * 已安裝 SOPS 工具。([https://github.com/mozilla/sops](https://github.com/mozilla/sops))
    * macOS (使用 Homebrew): `brew install sops`
    * 其他系統請參考 SOPS 官方文件。
    * 驗證：`sops --version`
* **3.4 SOPS 加密後端準備:**
    * 根據選擇的加密方式，完成對應設定：
        * **GPG:**
            * 已安裝 GnuPG (`gpg`).
            * 已生成或導入需要用於加密/解密的 GPG 金鑰對。
            * 知道相關人員的 GPG 金鑰指紋 (Fingerprint)。
        * **Cloud KMS (AWS/GCP/Azure):**
            * 已在對應雲平台上創建 KMS 金鑰。
            * 執行本機或 CI/CD 環境的用戶/服務帳號已被授予使用該 KMS 金鑰進行加密和解密的權限（例如，AWS IAM Policy, GCP IAM Role, Azure Role Assignment）。
            * 記下 KMS 金鑰的識別碼（如 AWS ARN, GCP Resource ID, Azure Key Vault Key ID）。
        * **age:**
            * 已安裝 age (`brew install age`).
            * 已生成 age 金鑰對。
            * 知道相關人員的 age 公鑰。

## 4. 標準作業程序 (Procedure)

**4.1 配置 SOPS 加密規則 (`.sops.yaml`)**

在你的 Helm Chart 或包含 Chart 的 Git 倉庫的 **根目錄** 下創建一個名為 `.sops.yaml` 的文件。此文件定義了 SOPS 如何加密該目錄下的文件。

* **目的:** 告知 SOPS 對哪些文件使用哪種加密方式和哪些金鑰。
* **創建 `.sops.yaml` 文件:**
    * **示例 1: 使用 GPG**
        ```yaml
        # .sops.yaml
        creation_rules:
          - path_regex: .*/secrets/.*\.yaml$ # 只加密 secrets/ 目錄下的 yaml 文件
            encrypted_regex: ^(data|stringData)$ # 加密 Secret object 中的 data/stringData (可選)
            pgp: 'FINGERPRINT_ALICE,FINGERPRINT_BOB,FINGERPRINT_CICD' # 替換成實際的 GPG 指紋，用逗號分隔
        ```
    * **示例 2: 使用 AWS KMS**
        ```yaml
        # .sops.yaml
        creation_rules:
          - path_regex: secrets\.yaml$ # 只加密名為 secrets.yaml 的文件
            kms: 'arn:aws:kms:us-east-1:123456789012:key/your-kms-key-id' # 替換成你的 KMS Key ARN
            # 可以添加多個 KMS 或混合其他，例如：
            # kms: 'arn:aws:kms:us-east-1:ACCOUNT_ID_1:key/KEY_ID_1,arn:aws:kms:eu-west-1:ACCOUNT_ID_2:key/KEY_ID_2'
            # pgp: 'FINGERPRINT_BACKUP_ADMIN' # 可以混合使用
        ```
    * **示例 3: 使用 age**
         ```yaml
        # .sops.yaml
        creation_rules:
          - path_regex: .*\.secret\.yaml$ # 加密所有以 .secret.yaml 結尾的文件
            age: 'age1_PUBLIC_KEY_ALICE,age1_PUBLIC_KEY_BOB,age1_PUBLIC_KEY_CICD' # 替換成實際的 age 公鑰
         ```
* **注意:**
    * `path_regex` 用於匹配需要加密的文件路徑。
    * `pgp`, `kms`, `age` 等欄位指定了加密使用的金鑰 ID 或識別符。
    * 確保所有需要解密權限的人/系統對應的金鑰或權限都已配置。

**4.2 創建敏感值文件**

將需要加密的敏感資料放入一個獨立的 YAML 文件。建議命名規範與 `.sops.yaml` 中的 `path_regex` 匹配。

* **示例:** 創建 `secrets.yaml` 或 `mychart/secrets/production.yaml`
    ```yaml
    # secrets.yaml
    database:
      username: dbuser
      password: "ThisWillBeEncrypted!" # 敏感資料
    apiCredentials:
      clientId: "client123"
      clientSecret: "SuperSecretOAuthToken" # 敏感資料
    ```

**4.3 使用 SOPS 加密敏感值文件**

使用 `helm secrets encrypt` 命令加密上一步創建的文件。`helm secrets` 會自動調用 SOPS，並根據 `.sops.yaml` 的規則進行加密。

* **執行加密:**
    ```bash
    helm secrets encrypt secrets.yaml
    ```
* **驗證:**
    * 加密成功後，`secrets.yaml` 文件的內容會變成包含 SOPS 元數據和密文的結構。
    * 直接 `cat secrets.yaml` 會看到類似以下的內容 (結構取決於後端)：
        ```yaml
        # 內容已被 SOPS 加密，無法直接讀取
        database:
            username: ENC[AES256_GCM,data:...=,...]
            password: ENC[AES256_GCM,data:...=,...]
        apiCredentials:
            clientId: ENC[AES256_GCM,data:...=,...]
            clientSecret: ENC[AES256_GCM,data:...=,...]
        sops:
            # ... SOPS 元數據，包含金鑰資訊、MAC 等 ...
        ```

**4.4 將加密文件提交到版本控制 (Git)**

* **添加加密文件:**
    ```bash
    git add secrets.yaml
    git add .sops.yaml # 同時提交 SOPS 配置文件
    ```
* **配置 `.gitignore` (重要):** 確保**永遠不會**提交未加密的敏感文件。
    ```gitignore
    # .gitignore

    # 忽略可能的明文秘密文件
    *.decrypted.yaml
    secrets*.plaintext.yaml
    # 如果你的加密文件有特定後綴 (例如 .enc.yaml)，可以忽略原始文件名
    # secrets.yaml

    # 根據你的命名習慣調整，確保只有加密後的文件被追蹤
    ```
    **最佳實踐:** 養成「先加密，再提交」的習慣，避免本地存在長時間的明文文件。加密通常是原地操作 (in-place)。

**4.5 在 Helm Chart 中引用加密值**

Helm 會將通過 `-f` 或 `--values` 傳遞的 Values 文件（包括 `helm secrets` 自動解密的文件）合併。你可以在 Chart 的模板中像引用普通值一樣引用加密文件中的值。

* **模板示例 (`templates/secret.yaml`):**
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: {{ include "mychart.fullname" . }}-db-creds
    type: Opaque
    stringData:
      # 引用來自 secrets.yaml 的值
      username: {{ .Values.database.username }}
      password: {{ .Values.database.password }}
    ```

**4.6 使用 `helm secrets` 部署或管理 Helm Release**

**務必** 使用 `helm secrets <command>` 替代標準的 `helm <command>`。

* **安裝 Chart:**
    ```bash
    helm secrets install my-release ./my-chart -f values.yaml -f secrets.yaml
    # 如果 secrets.yaml 在 Chart 目錄內且符合命名約定，有時可省略 -f secrets.yaml
    ```
* **升級 Chart:**
    ```bash
    helm secrets upgrade my-release ./my-chart -f values.yaml -f secrets.yaml
    ```
* **渲染模板 (調試):**
    ```bash
    helm secrets template my-release ./my-chart -f values.yaml -f secrets.yaml
    ```
* **查看 Values (解密後):**
    ```bash
    helm secrets get values my-release --all # 會顯示解密合併後的所有值
    ```
* **注意:** 如果未使用 `helm secrets` 前綴，Helm 將無法解密 `secrets.yaml`，導致模板渲染失敗或使用了錯誤的（空或預設）值。

**4.7 編輯已加密的 Secrets 文件**

使用 `helm secrets edit` 命令安全地編輯加密文件。

* **執行編輯:**
    ```bash
    helm secrets edit secrets.yaml
    ```
* **流程:**
    1.  `helm secrets` 調用 SOPS 解密 `secrets.yaml` 到一個臨時文件。
    2.  打開你的 `$EDITOR` (系統預設編輯器) 編輯該臨時文件。
    3.  保存並關閉編輯器。
    4.  `helm secrets` 調用 SOPS 使用 `.sops.yaml` 中的規則重新加密臨時文件，並覆蓋原始的 `secrets.yaml`。
* **提交變更:** 編輯完成後，將更新後的 `secrets.yaml` 提交到 Git。

**4.8 查看已加密 Secrets 文件的明文內容**

使用 `helm secrets view` 命令臨時查看解密後的內容，不會修改原文件。

* **執行查看:**
    ```bash
    helm secrets view secrets.yaml
    ```
* **輸出:** 文件內容將以明文形式打印到控制台。

## 5. 金鑰管理程序 (Key Management with SOPS)

這是確保流程安全的關鍵環節。

* **GPG:**
    * **共享:** 團隊成員需要安全地交換 GPG 公鑰。將所有需要解密權限者的公鑰指紋添加到 `.sops.yaml` 的 `pgp` 字段中。
    * **CI/CD:** 需要將用於解密的 CI/CD GPG **私鑰** 安全地存儲在 CI/CD 系統的 Secrets Manager 中，並在 Pipeline 執行時導入到 GPG 環境中。私鑰本身可能需要密碼保護。
    * **撤銷:** 如果成員離開或金鑰洩露，需要從 `.sops.yaml` 中移除對應指紋，並重新加密所有受影響的文件。
* **Cloud KMS (AWS/GCP/Azure):**
    * **權限:** 管理對應雲平台上的 IAM 策略/角色，確保只有授權的用戶和服務帳號（包括 CI/CD）擁有對 `.sops.yaml` 中指定的 KMS 金鑰的 `kms:Decrypt` (AWS) 或等效權限。
    * **CI/CD:** CI/CD 環境需要配置為使用具有解密權限的雲服務帳號憑證。
    * **審計:** 利用雲平台提供的 KMS 金鑰使用審計日誌來追踪解密操作。
    * **金鑰輪換:** 利用雲平台提供的自動金鑰輪換功能。SOPS 通常能處理好輪換後的金鑰。
* **age:**
    * **共享:** 團隊成員需安全地交換 age 公鑰。將公鑰添加到 `.sops.yaml` 的 `age` 字段。
    * **CI/CD:** 將 CI/CD 使用的 age **私鑰** 安全地存儲在 Secrets Manager 中，並在 Pipeline 中提供給 SOPS 使用。
    * **撤銷:** 從 `.sops.yaml` 移除公鑰並重新加密文件。

## 6. 故障排除 (Troubleshooting)

* **解密失敗:**
    * **原因:** 本地環境或 CI/CD 環境缺少所需的解密金鑰（GPG 私鑰、age 私鑰）或權限（KMS）。
    * **解決:** 檢查 GPG/age 金鑰是否已導入且可用，或雲平台 IAM 權限是否正確配置。確認 `.sops.yaml` 中指定的金鑰 ID/指紋/ARN 正確無誤。
* **忘記使用 `helm secrets` 前綴:**
    * **現象:** Helm 報錯，提示找不到值，或部署了包含加密佔位符（`ENC[...]`）的資源。
    * **解決:** 始終記得使用 `helm secrets install/upgrade/template` 等命令。
* **`.sops.yaml` 配置錯誤:**
    * **現象:** 加密失敗，或加密了非預期的文件/字段。
    * **解決:** 仔細檢查 `.sops.yaml` 中的 `path_regex` 和金鑰配置。

## 7. 參考資料 (References)

* **Helm Secrets 文檔:** [https://github.com/jkroepke/helm-secrets](https://github.com/jkroepke/helm-secrets)
* **Mozilla SOPS 文檔:** [https://github.com/mozilla/sops](https://github.com/mozilla/sops)
* **Helm 文檔:** [https://helm.sh/docs/](https://helm.sh/docs/)

## 8. 文件控制 (Document Control)

* **版本:** 1.0
* **制定日期:** 2025-04-24
* **制定者:** AI Assistant
* **審核者:** (待審核)