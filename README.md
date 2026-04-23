# Flow xử lý `app/repayment/usecase/interactor.go`

Tài liệu tóm tắt flow theo section spec `Nudge-バッチ仕様書_金利計算バッチ`. Mermaid render sẵn trên GitHub/GitLab.

---

## 1. Overview — Entry point

```mermaid
flowchart TD
    Start([main.go → Handler.Run]) --> Connect[Connect DB + ReadReplica<br/>handler.go]
    Connect --> Init[Khởi tạo repositories<br/>+ NewCalcInterest + NewInteractor]
    Init --> Retry{Retry loop<br/>max 3 lần}
    Retry --> CalcInterest[interactor.CalcInterest<br/>line 55]
    CalcInterest --> CheckRetry{retry == true?}
    CheckRetry -- Yes --> Sleep[sleep 500ms] --> Retry
    CheckRetry -- No --> End([return retValue])
```

---

## 2. `CalcInterest` — Batch entry (line 55)

```mermaid
flowchart TD
    A([CalcInterest start]) --> B[Tính month = time.Now.Month-2<br/>today = time.Now]
    B --> C[usersList hard-code<br/>hoặc ImportUsersListFile từ S3]
    C --> D{len usersList == 0?}
    D -- Yes --> E[Log: Calc user none<br/>retry = false]
    D -- No --> F[CalcLoop ctx, usersList, month, today, batchID]
    F --> G{len notImplementedUsersList == 0?}
    G -- Yes --> H[WriteFile empty UserList.FileName]
    G -- No --> I[WriteNotImplementedUsersList<br/>+ Upload S3]
    E --> Z([return retry, err])
    H --> Z
    I --> Z
```

---

## 3. `CalcLoop` — Main processing (line 303)

```mermaid
flowchart TD
    Start([CalcLoop start]) --> CommonRule[<b>NEW #820</b><br/>GetCommonMinRepaymentRule 1 lần<br/>line 304-318]
    CommonRule --> CR_Err{err?}
    CR_Err -- Yes --> Return1[return true, list, err]
    CR_Err -- No --> CR_Cache[cache commonBaseBillingAmount<br/>hasCommonBaseBilling flag]
    CR_Cache --> Loop{for cnt = 0<br/>cnt less than len userList}
    Loop -- Done --> EndOK([return retry, notImplList, retErr])
    Loop -- Next user --> User[userID = userList cnt .UserID<br/>creditLimit = strconv.ParseInt]
    User --> BeginTx[3.4.1 Begin Transaction]
    BeginTx --> Step35[3.5 GetSumCalcInterestRepayment<br/>month = 2 tháng trước]
    Step35 --> Step36[3.6 Flag batch ステータス<br/>delay / lost / overdraft]
    Step36 --> Step37[3.7 Repayment Status Judgment]
    Step37 --> Step38[3.8 Billing Amount Calc<br/>ghi nhánh a/b.0/b.1/b.2]
    Step38 --> Step39[3.9 Low-usage billing cancel]
    Step39 --> Step312[3.12 Interest Calc]
    Step312 --> Step313[3.13 Update repayment_balances<br/>total + interest_target]
    Step313 --> Commit[Commit Transaction]
    Commit --> Loop
```

---

## 4. Section 3.6–3.7 — Status gathering & judgment

```mermaid
flowchart TD
    Start([After 3.5]) --> S361[3.6.1 GetRepaymentStatuses<br/>IDs 001/002/003/013/014]
    S361 --> S362[3.6.2 GetRepaymentStatusNames]
    S362 --> S363[3.6.3 Update item check<br/>collect bool flags]
    S363 --> S364[3.6.4 UpdateRepaymentStatusControl]
    S364 --> S371[3.7.1 Re-fetch statuses]
    S371 --> S372[3.7.2 GetMonthlyBillingAmount<br/>where repaid_flg=0]
    S372 --> S373[3.7.3 GetMonthlyOdBillingAmount]
    S373 --> S374[3.7.4 Status transition]
    S374 --> B1{hasEntrustment<br/>or hasSuspension?}
    B1 -- Yes --> B1Path[B.1 期失解除 check<br/>B.2 遅延解除<br/>B.3 OD解除]
    B1 -- No --> A1[3.7.4.A.1 Delay check<br/>A.2 Lost/Entrustment check<br/>A.3 Overdraft check]
    B1Path --> S375
    A1 --> S375[3.7.5 Update control flags again]
    S375 --> S38([to 3.8])
```

---

## 5. Section 3.8 — 請求額判定 (Billing Judgment) — Nhánh bị thay đổi #820

```mermaid
flowchart TD
    Start([3.8 Start]) --> UserRule[<b>NEW #820</b><br/>GetUserMinRepaymentRule<br/>line 1849-1868]
    UserRule --> UR_Err{err?}
    UR_Err -- Yes --> Rollback1[Rollback + retry]
    UR_Err -- No --> RulePri{User rule > 0?}
    RulePri -- Yes --> UseUser[baseBillingAmount = userRule.MinRepaymentAmount]
    RulePri -- No --> ChkCommon{hasCommonBaseBilling?}
    ChkCommon -- Yes --> UseCommon[baseBillingAmount = commonBaseBillingAmount]
    ChkCommon -- No --> Err204[<b>NEW #820</b><br/>err = Wrap1 900801000900204<br/>Rollback + retry]
    
    UseUser --> StopChk{stop_billing_flg == 1?}
    UseCommon --> StopChk
    StopChk -- Yes --> BranchA[3.8.B.2.A<br/>repaidFlg = 1<br/>repaidOdFlg = 1<br/>Patch monthly_billing/od]
    StopChk -- No --> CalcOD[newODBillingAmount =<br/>total_repayment_balance<br/>- credit_limit<br/>- sumNewOdBilling-sumRepaidOdBiling<br/>clamp >= 0]
    
    CalcOD --> AccTxfer{accountTranferFlg == 1?}
    AccTxfer -- Yes --> BranchB0[b.0 口座振替<br/>newODBilling = 0<br/>newTotalBilling = total-sumBill<br/>newNormalBilling = newTotalBilling]
    AccTxfer -- No --> HasLost{hasLostStatus?}
    HasLost -- Yes --> BranchB1[b.1 期失<br/>newTotalBilling = total-sumBill<br/>newNormalBilling = newTotal - newOD]
    HasLost -- No --> B2Cond{diff >= baseBillingAmount?<br/><b>#820: dynamic</b>}
    B2Cond -- Yes --> B21[<b>b.2.1 — MODIFIED #820</b><br/>newNormalBilling = min baseBilling, diff<br/>newTotalBilling = newOD + newNormal]
    B2Cond -- No --> B22[<b>b.2.2 — COMMENTED #820</b><br/>分岐不要<br/>giữ newTotalBilling = 0<br/>newNormalBilling = 0]
    
    BranchA --> Step383
    BranchB0 --> Step383
    BranchB1 --> Step383
    B21 --> Step383
    B22 --> Step383
    Step383[3.8.B.3 Insert records<br/>monthly_billing_amounts<br/>monthly_od_billing_amounts<br/>monthly_billing_histories]
    Step383 --> Out([to 3.9])
```

---

## 6. Section 3.9 — 少額ユーザ 請求取消

```mermaid
flowchart TD
    Start([3.9 start]) --> AccChk{accountTranferFlg == 0?}
    AccChk -- No --> Skip([skip → 3.12])
    AccChk -- Yes --> Cond1{hasDelay AND NOT hasOverdraft AND NOT hasLost?}
    Cond1 -- Yes --> C1_Diff{sumRepayment - sumRepaid < 1000?}
    C1_Diff -- Yes --> Patch1[PatchRepaymentStatuses02<br/>遅延解除]
    C1_Diff -- No --> Cond2
    Cond1 -- No --> Cond2
    Patch1 --> Cond2{hasOverdraft AND NOT hasLost<br/>AND sumRepayment-sumRepaid<1000<br/>AND total_repayment_balance<1000?}
    Cond2 -- Yes --> Patch2[PatchRepaymentStatuses02<br/>オーバードラフト解除]
    Cond2 -- No --> End([→ 3.12])
    Patch2 --> End
```

---

## 7. Section 3.12 — 利息計算 (Interest Calc)

```mermaid
flowchart TD
    Start([3.12 entry]) --> G1{stopCalcInterestFlg == 0<br/>AND accountTranferFlg == 0?}
    G1 -- No --> Skip([skip → 3.13])
    G1 -- Yes --> GetRates[GetInterestRateInID ids=1,3]
    GetRates --> RatesErr{rates == nil OR err?}
    RatesErr -- Yes --> Rollback[Wrap 900801000900204<br/>Rollback + retry]
    RatesErr -- No --> Diff[diff = sumRepayment - sumRepaid]
    
    Diff --> Range{Rate range?}
    Range -- lower < diff < upper --> Calc1[rate = interestRateId1<br/>oneDayInterest = rate × diff-sumBill]
    Range -- diff <= lower --> Zero[oneDayInterest = 0]
    Range -- diff >= upper --> Calc3[rate = interestRateId3<br/>oneDayInterest = rate × diff-sumBill]
    
    Calc1 --> IntChk{oneDayInterest >= 1?}
    Calc3 --> IntChk
    Zero --> End
    IntChk -- Yes --> GetTotal[GetTotalInterests]
    GetTotal --> Exists{len totalInterests == 0?}
    Exists -- Yes --> PutTotal[PutTotalInterest<br/>3.12.4.1]
    Exists -- No --> UpdTotal[UpdateTotalInterest<br/>3.12.5]
    PutTotal --> PutDetail[PutInterestDetails]
    UpdTotal --> PutDetail
    PutDetail --> IntChk2{err?}
    IntChk2 -- Yes --> Rollback
    IntChk2 -- No --> End
    IntChk -- No --> End([→ 3.13])
```

---

## 8. Section 3.13 — Update `repayment_balances`

```mermaid
flowchart TD
    Start([3.13 entry]) --> Calc[preInterestTarget =<br/>repayment_balance<br/>- sumRepayment - sumRepaid]
    Calc --> Clamp{oneDayInterest <= 0?}
    Clamp -- Yes --> ZeroInt[oneDayInterest = 0]
    Clamp -- No --> Total
    ZeroInt --> Total[totalRepaymentBalance =<br/>current total_repayment_balance<br/>+ oneDayInterest<br/>+ oneDayDelayInterest]
    Total --> Update[UpdateRepaymentBalance<br/>total + interest_target + pre_interest_target]
    Update --> Err{err?}
    Err -- Yes --> Rollback[Rollback + retry continue]
    Err -- No --> Commit[Commit Transaction]
    Commit --> Loop([→ next user])
```

---

## 9. Error handling pattern (chung cho mọi step)

```mermaid
flowchart TD
    Step[Step X: DB call / business logic] --> ChkErr{err != nil?}
    ChkErr -- No --> Next([next step])
    ChkErr -- Yes --> SetUser[ctx = SetUserID ctx, userID]
    SetUser --> RB[Rollback transaction]
    RB --> RBErr{rollbackErr?}
    RBErr -- Yes --> Fatal[FatalErrHundling<br/>return false, list, err]
    RBErr -- No --> Warn[apllog.Warnf LW0100000102]
    Warn --> Continue[ContinueErrHundling<br/>append to notImplementedUsersList<br/>retry = true<br/>continue loop]
```

---

## 10. Retry strategy

```mermaid
flowchart LR
    H[Handler.Run<br/>for cnt=0; cnt<3; cnt++] --> CI[CalcInterest]
    CI --> R{retry == true<br/>err != nil?}
    R -- retry=true --> Sleep[500ms]
    Sleep --> H
    R -- retry=false, err=nil --> Exit0[retValue = 0<br/>break]
    R -- retry=false, err!=nil --> Exit1[retValue = 1<br/>break]
```

---

## 11. Tổng thể end-to-end (zoom out)

```mermaid
flowchart TB
    subgraph Handler
        H1[Handler.Run] --> H2[Connect DB] --> H3[Build deps] --> H4[Retry loop x3]
    end
    subgraph Interactor
        I1[CalcInterest] --> I2[Load user list]
        I2 --> I3[CalcLoop]
    end
    subgraph CalcLoop
        direction TB
        L0[<b>#820</b> GetCommonMinRepaymentRule] --> L1[for user in userList]
        L1 --> L2[Begin Tx]
        L2 --> L3[3.5 sum balances]
        L3 --> L4[3.6 status gather]
        L4 --> L5[3.7 status judge]
        L5 --> L6[3.8 billing calc<br/>bao gồm <b>#820</b> GetUserMinRepaymentRule<br/>+ b.2.1 min base, diff]
        L6 --> L7[3.9 small-amount cancel]
        L7 --> L8[3.12 interest calc]
        L8 --> L9[3.13 update balance]
        L9 --> L10[Commit]
        L10 --> L1
    end
    H4 --> I1
    I3 --> CalcLoop
    L1 -. done .-> Final[notImplementedUsersList → S3]
    Final --> End([return])
```

---

## 12. Điểm thay đổi của Issue #820 (highlighted)

| Nơi | Trước | Sau |
|---|---|---|
| Đầu `CalcLoop` (line 304-318) | — | Fetch `common_min_repayment_rules` **1 lần** / batch |
| Trong for-user (line 1849-1899) | Hard-code `1000` | Fetch `user_min_repayment_rules` → priority user > common > **error 900801000900204** |
| `b.2.1` (line 2018-2037) | `newNormalBilling = 1000` | `newNormalBilling = min(baseBillingAmount, diff)` |
| `b.2.2` (line 2039-2050) | `newTotalBilling = newODBilling`<br/>`newNormalBilling = 0` | **Comment out** (分岐不要) — giữ default 0 |

```mermaid
flowchart LR
    subgraph Before[Code cũ]
        BA[hard-code 1000]
        BB[b.2.1: newNormal = 1000]
        BC[b.2.2: newTotal = newOD<br/>when diff < 1000]
    end
    subgraph After[Code mới #820]
        AA[common rule fetch 1 lần]
        AB[user rule fetch per user<br/>priority user > common > ERROR]
        AC[b.2.1: min baseBilling, diff]
        AD[b.2.2: COMMENTED]
    end
    Before -. #820 .-> After
```
