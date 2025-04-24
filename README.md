# Helm Secrets 使用方法與詳細說明

#### Helm Secrets 是 Helm 的一個插件，主要用於安全地管理 Kubernetes 部署過程中的敏感資訊（如密碼、API 金鑰等），並且能與 Git 工作流無縫整合。其核心是結合 Mozilla SOPS（Secrets OPerationS）工具，實現對 YAML/JSON 等格式的加密與解密。以下將詳細整理其安裝、基本原理、實作步驟與常見應用場景，方便你做為筆記參考。

## 一、核心原理與功能
### 敏感資訊加密：
將 Helm Chart 需用到的 secrets 檔案（如 secrets.yaml）加密，安全存放於 Git。

### 動態解密：
部署時自動解密，確保敏感資訊只在需要時暴露於記憶體。

### 多後端支援：
可結合 SOPS 支援的 PGP、AWS KMS、GCP KMS、Azure Key Vault 等多種金鑰管理方式。

### GitOps 友善：適合與 ArgoCD、Flux 等自動化部署工具整合。

# 二、安裝步驟
### 安裝 Helm Secrets 插件

```
helm plugin install https://github.com/jkroepke/helm-secrets
```
### 安裝 SOPS 工具


#### 直接下載二進制檔（選版本）
```
wget https://github.com/mozilla/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
sudo mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops
```

#### 或使用 snap（若系統支援）
```
sudo snap install sops
```

### 若用 PGP/GPG 加密：
```
sudo apt install gnupg
gpg --full-generate-key  # 生成金鑰對
gpg --list-secret-keys   # 查詢 fingerprint

# AWS KMS（需預先設定 AWS CLI 權限）
# GCP KMS（需安裝 gcloud CLI）
```

## 三、加密流程（以 GPG 為例）
### 產生 GPG 金鑰對
```
gpg --full-generate-key
```
#### 依指示輸入姓名、Email、密碼等資訊。

### 建立 .sops.yaml 規則檔

#### 指定哪些欄位要加密、使用哪個金鑰 fingerprint。
範例內容：
```
creation_rules:
- path_regex: secrets\..*\.yaml$
  key_groups:
    - pgp: <YOUR_GPG_FINGERPRINT>
```    
### 撰寫要加密的 secrets 檔案

例如 secrets.prod.yaml，內容如：
```
API_KEY: my-secret-key
DB_PASSWORD: my-db-password
```
### 加密檔案
```
helm secrets enc secrets.prod.yaml
```
產生加密後的 secrets.prod.yaml（內容已被 SOPS 加密）。

## 四、解密與部署流程
### 解密檔案（本地測試或除錯）

```
helm secrets dec secrets.prod.yaml
```
會產生一個暫時的明文檔案 secrets.prod.yaml.dec。

### 部署 Chart（自動解密）

```
helm secrets install my-release ./my-chart -f secrets.prod.yaml
```
在部署時，Helm Secrets 會自動解密 secrets 檔案並傳遞給 Helm。

### 升級/回滾等操作也同理

```
helm secrets upgrade my-release ./my-chart -f secrets.prod.yaml
```
## 五、常見應用場景
### 團隊協作：
所有敏感資訊都以加密狀態存放於 Git，團隊成員可安全協作。
### CI/CD 自動化：
配合 GitOps 工具（如 ArgoCD），自動解密並部署，無需人工干預。

### 多雲環境整合：
可根據不同環境選擇不同的 KMS 後端，靈活應對各種安全需求。

## 六、進階補充
### 只加密部分欄位：
可透過 .sops.yaml 規則，僅加密 YAML 中特定 key（如 password、token）。

### 與 vals 整合：
若需從外部來源（如 AWS SecretManager）動態拉取 secrets，可結合 vals 工具。

### Git Diff 支援：
安裝後會自動設定 Git diff，方便檢視加密檔案的變更。

## 七、常用指令速查
### 指令	功能說明
```
helm secrets enc <file>	加密 secrets 檔案
helm secrets dec <file>	解密 secrets 檔案
helm secrets view <file>	只讀取解密內容（不生成檔案）
helm secrets install ...	安裝時自動解密
helm secrets upgrade ...	升級時自動解密
```
## 八、注意事項
- 加密檔案（如 secrets.prod.yaml）才應 push 到 Git，解密檔案（如 .dec）請勿上傳。

- 金鑰管理需謹慎，遺失私鑰將無法解密。

- 若用雲端 KMS，請確保 CI/CD pipeline 有對應權限。