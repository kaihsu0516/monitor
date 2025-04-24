# Helm Secrets 使用方法詳解

## 1. 簡介 (Introduction)

`helm secrets` 是一個 Helm 插件，它允許你對 Helm Chart 中的敏感資料（例如密碼、API 金鑰）進行加密，並將加密後的文件安全地存儲在版本控制系統（如 Git）中。當你使用 Helm 部署 Chart 時，`helm secrets` 會在需要時自動解密這些敏感資料，將其注入到 Kubernetes 中，而無需在 CI/CD 流程或本地環境中暴露明文密碼。

**為什麼需要 Helm Secrets？**

-   **安全性 (Security):** 避免將明文密碼、金鑰等敏感資訊直接提交到版本控制系統中，降低洩漏風險。
-   **版本控制 (Version Control):** 可以像管理其他程式碼一樣，對包含敏感資訊的配置文件進行版本追踪和管理。
-   **GitOps 整合 (GitOps Integration):** 非常適合 GitOps 工作流程，因為所有配置（包括加密的敏感資訊）都存儲在 Git 中。
-   **協作 (Collaboration):** 團隊成員可以共享包含加密敏感資訊的 Chart，只要他們擁有解密所需的金鑰。

## 2. 先決條件 (Prerequisites)

在使用 `helm secrets` 之前，你需要確保：

1.  **已安裝 Helm:** `helm secrets` 是 Helm 的插件，所以你需要先安裝 Helm v3。
2.  **選擇並安裝加密後端:** `helm secrets` 本身不執行加密，它依賴於外部的加密工具。最常見的後端包括：
    * **GnuPG (GPG):** 一種廣泛使用的開源加密標準。你需要安裝 GPG 並生成或導入 GPG 金鑰對。
    * **Mozilla SOPS (Secrets OPerationS):** 一個更現代化的工具，可以與多種加密服務集成，如 GPG、age、AWS KMS、GCP KMS、Azure Key Vault 等。推薦使用 SOPS，因為它更靈活。
    * **特定雲提供商 KMS (AWS KMS, GCP KMS, Azure Key Vault):** 如果你主要使用某個雲平台，可以直接利用其提供的金鑰管理服務。通常需要配合 SOPS 使用。

## 3. 加密後端選項 (Encryption Backend Options)
- GPG:
  - 需要安裝 GPG 工具鏈。
  - 需要生成或導入 GPG 金鑰對。
  - 通過 .gnupg 目錄或環境變數管理金鑰。
  - 加密時需要指定接收者的公鑰指紋（fingerprint）。
  - 解密時需要本地 GPG 環境中有對應的私鑰和密碼（如果私鑰有密碼）。
- SOPS:
  - 推薦！更為靈活和強大。
  - 需要安裝 SOPS。
  - 通過一個 ```.sops.yaml``` 文件在項目根目錄定義加密規則（使用哪個後端，哪些金鑰 ID 等）。
  - 支持多種後端：GPG, age, AWS KMS, GCP KMS, Azure Key Vault。
  - 可以為不同的文件或項目部分配置不同的加密規則。
   
## 4. 金鑰管理 (Key Management)
這是使用 ```helm secrets``` 最關鍵的部分：

- 私鑰安全: 無論使用 GPG 還是雲 KMS，解密所需的私鑰或訪問權限必須得到妥善保護。
- 公鑰/權限共享:
  - GPG: 團隊成員需要交換 GPG 公鑰，以便將所有需要解密的人的公鑰添加到加密文件的接收者列表中。CI/CD 環境也需要導入用於解密的 GPG 私鑰（通常存儲在 CI/CD 系統的 Secrets Management 中）。
  - SOPS + KMS: 團隊成員和 CI/CD 環境需要被授予對應雲平台 KMS 金鑰的解密權限（例如，通過 IAM 角色或策略）。
- 金鑰輪換: 定期輪換加密金鑰是一種良好的安全實踐。
## 5. 優點 (Benefits)
- Secrets-in-Git: 安全地將敏感配置與應用程式碼一起存儲在 Git 中。
- 審計追踪: 所有配置（包括加密的敏感資訊）的變更都有 Git 歷史記錄。
- 簡化部署: CI/CD 流程只需要訪問解密金鑰或權限，無需管理大量明文環境變數。
- 一致性: 開發、測試和生產環境可以使用相同的流程來管理敏感資料。
## 6. 注意事項 (Considerations)
- 金鑰管理複雜性: 需要建立一套安全的流程來管理和分發加密/解密所需的金鑰或權限。
- CI/CD 集成: 需要在 CI/CD 環境中安全地配置解密所需的憑證（GPG 私鑰、雲服務帳戶權限等）。
- 開發者設置: 團隊中的每個開發者都需要配置好本地的解密環境（安裝 GPG/SOPS，導入金鑰等）。
- 命令差異: 必須記得使用 ```helm secrets <command>``` 而不是 ```helm <command>``` 來處理包含加密文件的 Chart。
## 7. 結論 (Conclusion)
```helm secrets``` 是一個強大的 Helm 插件，通過與 GPG 或 SOPS 等加密工具集成，解決了在版本控制系統中安全管理 Helm Chart 敏感資料的痛點。它使得實現 Secrets-in-Git 和遵循 GitOps 實踐變得更加容易，但同時也引入了對金鑰管理的依賴。對於需要安全、可追踪地管理 Kubernetes 配置密碼的團隊來說，```helm secrets``` 是一個非常有價值的工具。

## Ref:
請參考 [Helm Secrets 使用方法](helm_secrets.md) 進行設定。