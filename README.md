# LBARS - Load Balancer with Adaptive Recovery Sentry

## 功能規劃
- 1 分鐘健康檢查 / 10 分鐘故障恢復 / 5 分鐘 MaxConn 動態調整
- 請求時間 + Ping延遲 + 響應速度 + LLM 延遲加權計算
- 連線池
- 記憶體預分配
- 動態調整 MaxConn
- 故障/超時立即降級，0 延遲重試下個後端
- proxypass 處理請求，timer 處理健康檢查
- 提供 healthPath / llmCheckPath 參數用於檢查端點健康與 LLM 響應速度
- 最小 token 測量 API 延遲
- Email 故障通知

## 指令規劃
- `lbars start --port {PORT}`<br>
    - 尚未啟動實例：啟動一個 {PORT} 的實例
    - 已有實例，詢問「目前已經有｛當前PORT｝的實例，是否更換 PORT 至 8000？」
        - `yes / y`：重啟並更換 PORT
        - `no`（預設）：不做任何動作
- `lbars stop`<br>
    關閉當前實例，如果當前沒實例則略過
- `lbars status`<br>
    當前實例的健康狀態
- `lbars monit`<br>
    顯示狀態（類似 pm2 monit）
- `lbars add`<br>
    增加後端端點，依序詢問以下問題
    1. 後端位置？
    2. 健康檢查路徑？
    3. llm 推理路徑？
- `lbars list`<br>
    列出全部後端
- `lbars rm {ID}`<br>
    移除後端
- `lbars flush`<br>
    強制健康檢查

## monit 區塊規劃（2x2）
- 上左: 後端端點的即時健康狀態
- 上右: 日誌輸出
- 下左: 後端端點的 config
- 下右: 指令輸入區塊（可以直接下 lbars 指令）

## 流程
```mermaid
graph TD
 A[請求] --> B{健康列表是否為空}
 B -->|是<br>503| Z[請求完成]
 B -->|否| B2[選擇後端]
 
 B2 --> C{檢查 MaxConn}
 C -->|Conn < MaxConn<br>conn++| C1{15秒內收到響應?}
 C -->|Conn >= MaxConn<br>下一個| D
 
 C1 -->|成功<br/>conn--| C11[轉發至客戶端]
 C1 -->|超時/錯誤<br/>立即降級| C12[移至故障列表]
 
 C12 --> D{健康列表為空?}
 D -->|否<br>下一個| B2
 D -->|是<br>503| Z
 
 C11 --> Z
 
 JJ -.-> B2
 NN -.-> B2
 
 subgraph "背景健康檢查系統"
 DD[健康檢查<br/>1分鐘]
 EE[故障檢查<br/>10分鐘]
 GG[動態計分<br/>5分鐘]
 
 DD -->|併發| HH{健康狀態<br>若有 llmCheckPath 則也必須有回應}
 HH -->|正常| II[維持健康]
 HH -->|故障| JJ[移至故障列表]
 
 EE -->|併發| LL{健康狀態<br>若有 llmCheckPath 則也必須有回應}
 LL -->|正常| NN[移回健康列表]
 LL -->|故障| OO[保持故障狀態]
 
 GG --> PP{效能評分<br>可設置最高最低}
 PP -->|快| QQ[MaxConn +10]
 PP -->|慢| RR[MaxConn -5]
 end
```
