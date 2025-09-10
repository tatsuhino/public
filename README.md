```


# Azure パラメータ設定一覧

## 1. リソース設定

### App Service

| パラメータ                     | Staging (開発) | Production (本番)           | 説明                                       |
| ------------------------------ | -------------- | --------------------------- | ------------------------------------------ |
| **SKU**                        | B1             | B1                          | App Service Plan のサイズ（Basic Tier）    |
| **Health Check Eviction Time** | 5 分           | 5 分                        | ヘルスチェック失敗時のエビクション時間     |
| **Auto-scaling**               | 無効           | 最小 1、最大 2 インスタンス | CPU 70%でスケールアウト、30%でスケールイン |

### PostgreSQL Database

| パラメータ               | Staging (開発)  | Production (本番) | 説明                            |
| ------------------------ | --------------- | ----------------- | ------------------------------- |
| **SKU**                  | B_Standard_B1ms | B_Standard_B1ms   | データベースサーバーのサイズ    |
| **Storage Size**         | 32768 MB (32GB) | 32768 MB (32GB)   | ストレージサイズ                |
| **High Availability**    | 無効            | 無効※             | 高可用性設定（※設定可能）       |
| **Backup Retention**     | 7 日            | 7 日              | PostgreSQL バックアップ保持期間 |
| **Geo-redundant Backup** | 無効            | 無効              | 地理冗長バックアップ            |

## 3. ログ保存期間設定

### 統合ログ管理一覧

| ログ種別        | ログ名               | 保存先 1         | 保持期間 1 | 保存先 2                  | 保持期間 2 | 保存先 3 | 保持期間 3 | 備考                            |
| --------------- | -------------------- | ---------------- | ---------- | ------------------------- | ---------- | -------- | ---------- | ------------------------------- |
| **App Service** | HTTP ログ            | ファイルシステム | 7 日       | Blob Storage (logs)       | 365 日     | -        | -          | Web アクセスログ                |
| **App Service** | アプリケーションログ | ファイルシステム | -          | Blob Storage (logs)       | 365 日     | -        | -          | アプリケーション実行ログ        |
| **App Service** | コンソールログ       | Log Analytics    | 30 日      | Blob Storage (log-export) | 365 日※    | -        | -          | Docker コンテナの stdout/stderr |
| **PostgreSQL**  | PostgreSQL ログ      | Log Analytics    | 30 日      | -                         | -          | -        | -          | データベース実行ログ            |

## 4. 監視・アラート設定

### メトリックアラート

| アラート名          | 対象リソース      | 閾値 | 監視間隔 | 説明                  |
| ------------------- | ----------------- | ---- | -------- | --------------------- |
| **App Service CPU** | App Service Plan  | 80%  | 5 分     | CPU 使用率監視        |
| **Web App Memory**  | Web App           | 1GB  | 5 分     | メモリ使用量監視      |
| **PostgreSQL CPU**  | PostgreSQL Server | 80%  | 1 分     | データベース CPU 監視 |


# =============================================================================
# Log Analytics Data Export - ログをStorage Accountに継続的にエクスポート
# =============================================================================

# Log Analytics WorkspaceからStorage Accountへのデータエクスポート設定
# コンテナログを長期保存用ストレージに自動転送
resource "azurerm_log_analytics_data_export_rule" "container_logs" {
  name                    = "${var.project}-${var.environment}-container-logs-export"
  resource_group_name     = data.azurerm_resource_group.main.name
  workspace_resource_id   = azurerm_log_analytics_workspace.main.id
  destination_resource_id = azurerm_storage_account.main.id
  enabled                 = true

  # エクスポートするテーブル（コンテナログのみ）
  table_names = [
    "AppServiceConsoleLogs_CL",  # Dockerコンテナのstdout/stderrログ
    "AppServiceHTTPLogs_CL",     # HTTPアクセスログ
    "AppServiceAppLogs_CL"       # アプリケーションログ
  ]

  # エクスポート先のStorage Account内のコンテナ名
  # 形式: am-{workspace-name}/{table-name}/y={year}/m={month}/d={day}/h={hour}/m={minute}/
}
  
# Data Export用のStorage Accountコンテナ
# エクスポートされたログファイルが自動作成されるコンテナ
resource "azurerm_storage_container" "log_export" {
  name                  = "logs"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}


```
