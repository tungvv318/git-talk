# Flow xử lý `app/repayment/usecase/interactor.go`

Tài liệu tóm tắt flow theo section spec `Nudge-バッチ仕様書_金利計算バッチ`, có annotate **table nguồn** cho từng bước.

> 📥 = READ (query), 📤 = WRITE (insert/update/delete).

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
    B --> C[usersList hard-code<br/>hoặc ImportUsersListFile từ S3<br/>📥 S3: UserList file]
    C --> D{len usersList == 0?}
    D -- Yes --> E[Log: Calc user none<br/>retry = false]
    D -- No --> F[CalcLoop ctx, usersList, month, today, batchID]
    F --> G{len notImplementedUsersList == 0?}
    G -- Yes --> H[WriteFile empty UserList.FileName<br/>📤 S3]
    G -- No --> I[WriteNotImplementedUsersList<br/>+ Upload S3<br/>📤 S3]
    E --> Z([return retry, err])
    H --> Z
    I --> Z
```

---

## 3. `CalcLoop` — Main processing (line 303)

```mermaid
flowchart TD
    Start([CalcLoop start]) --> CommonRule[<b>NEW #820</b> line 304-318<br/>GetCommonMinRepaymentRule<br/>📥 common_min_repayment_rules<br/>WHERE start_month <= currentMonth<br/>ORDER BY start_month DESC LIMIT 1]
    CommonRule --> CR_Err{err?}
    CR_Err -- Yes --> Return1[return true, list, err]
    CR_Err -- No --> CR_Cache[cache commonBaseBillingAmount = MinRepaymentAmount<br/>hasCommonBaseBilling flag]
    CR_Cache --> Loop{for cnt = 0<br/>cnt less than len userList}
    Loop -- Done --> EndOK([return retry, notImplList, retErr])
    Loop -- Next user --> User[userID = userList cnt .UserID<br/>creditLimit = strconv.ParseInt<br/>📥 userList from param / S3]
    User --> BeginTx[3.4.1 Begin Transaction]
    BeginTx --> Step34[3.4 GetRepaymentBalance<br/>📥 repayment_balances<br/>WHERE user_id = ? FOR UPDATE]
    Step34 --> Step35[3.5 GetSumCalcInterestRepayment<br/>📥 monthly_repayment_balances<br/>SUM repayment_balance, SUM repayed_balance<br/>WHERE user_id=? AND date_of_use<=month AND repayed_flg=0]
    Step35 --> Step352[3.5-2 GetDirectDebitPaymentTerm<br/>📥 direct_debit_payment_terms<br/>WHERE apply_start_date<=batchDate<br/>AND apply_end_date>batchDate AND status=2<br/>→ accountTranferFlg]
    Step352 --> Step36[3.6 Status init]
    Step36 --> Step37[3.7 Status judgment]
    Step37 --> Step38[3.8 Billing calc]
    Step38 --> Step39[3.9 Low-usage cancel]
    Step39 --> Step312[3.12 Interest calc]
    Step312 --> Step313[3.13 Update balance]
    Step313 --> Commit[Commit Transaction]
    Commit --> Loop
```

---

## 4. Section 3.6 – 3.7 — Status gathering & judgment

```mermaid
flowchart TD
    Start([After 3.5]) --> S361[3.6.1 GetRepaymentStatuses<br/>📥 repayment_statuses<br/>WHERE user_id=? AND status_id in 001/002/003/013/014]
    S361 --> S362[3.6.2 GetRepaymentStatusNames<br/>📥 repayment_status_names<br/>master data]
    S362 --> S363[3.6.3 Update item check<br/>collect bool flags]
    S363 --> S364[3.6.4 Update / Insert repayment_status_controls<br/>📤 repayment_status_controls]
    S364 --> S371[3.7.1 Re-fetch statuses<br/>📥 repayment_statuses]
    S371 --> S372[3.7.2 GetMonthlyBillingAmount<br/>📥 monthly_billing_amounts<br/>WHERE user_id=? AND repaid_flg=0<br/>ORDER BY billing_month asc]
    S372 --> S373[3.7.3 GetMonthlyOdBillingAmount<br/>📥 monthly_od_billing_amounts<br/>WHERE user_id=? AND repaid_flg=0]
    S373 --> S374[3.7.4 Status transition<br/>compute hasDelay/hasLost/hasOverdraft<br/>sumNewBilling / sumRepaidBiling<br/>sumNewOdBilling / sumRepaidOdBiling]
    S374 --> B1{hasEntrustment<br/>or hasSuspension?}
    B1 -- Yes --> B1Path[B.1 期失解除<br/>B.2 遅延解除<br/>B.3 OD解除<br/>📤 repayment_statuses applied_flg=0<br/>📤 repayment_status_histories]
    B1 -- No --> A1[3.7.4.A.1 Delay check<br/>A.2 Lost/Entrustment check<br/>A.3 Overdraft check<br/>📥 acceleration_notice_dates<br/>📤 repayment_statuses insert new<br/>📤 repayment_status_histories]
    B1Path --> S375
    A1 --> S375[3.7.5 Update control flags again<br/>📥 repayment_statuses<br/>📥 repayment_status_names<br/>📤 repayment_status_controls]
    S375 --> S38([to 3.8])
```

---

## 5. Section 3.8 — 請求額判定 (Billing Judgment) — Nhánh bị thay đổi #820

```mermaid
flowchart TD
    Start([3.8 Start]) --> UserRule[<b>NEW #820</b> line 1849-1868<br/>GetUserMinRepaymentRule<br/>📥 user_min_repayment_rules<br/>WHERE user_id=? AND start_month<=currentMonth<br/>ORDER BY start_month DESC LIMIT 1]
    UserRule --> UR_Err{err?}
    UR_Err -- Yes --> Rollback1[Rollback + retry]
    UR_Err -- No --> RulePri{User rule > 0?}
    RulePri -- Yes --> UseUser[baseBillingAmount = userRule.MinRepaymentAmount<br/>📥 user_min_repayment_rules]
    RulePri -- No --> ChkCommon{hasCommonBaseBilling?}
    ChkCommon -- Yes --> UseCommon[baseBillingAmount = commonBaseBillingAmount<br/>📥 common_min_repayment_rules cache]
    ChkCommon -- No --> Err204[<b>NEW #820</b><br/>err = Wrap1 900801000900204<br/>Rollback + retry]
    
    UseUser --> StopChk{stop_billing_flg == 1?<br/>📥 repayment_status_controls}
    UseCommon --> StopChk
    StopChk -- Yes --> BranchA[3.8.B.2.A<br/>repaidFlg = 1<br/>repaidOdFlg = 1<br/>📤 monthly_billing_amounts PATCH<br/>📤 monthly_od_billing_amounts PATCH]
    StopChk -- No --> CalcOD[newODBillingAmount =<br/>repayment_balances.total_repayment_balance<br/>- credit_limit users<br/>- sumNewOdBilling-sumRepaidOdBiling<br/>clamp >= 0]
    
    CalcOD --> AccTxfer{accountTranferFlg == 1?<br/>📥 direct_debit_payment_terms}
    AccTxfer -- Yes --> BranchB0[b.0 口座振替<br/>newODBilling = 0<br/>newTotalBilling = total-sumBill<br/>newNormalBilling = newTotalBilling]
    AccTxfer -- No --> HasLost{hasLostStatus?<br/>📥 repayment_statuses id=003}
    HasLost -- Yes --> BranchB1[b.1 期失<br/>newTotalBilling = total-sumBill<br/>newNormalBilling = newTotal - newOD]
    HasLost -- No --> B2Cond{diff >= baseBillingAmount?<br/>diff = sumRepayment-sumRepaid<br/>- sumNewBilling-sumRepaidBiling<br/><b>#820: dynamic</b>}
    B2Cond -- Yes --> B21[<b>b.2.1 — MODIFIED #820</b><br/>newNormalBilling = min baseBilling, diff<br/>newTotalBilling = newOD + newNormal]
    B2Cond -- No --> B22[<b>b.2.2 — COMMENTED #820</b><br/>分岐不要<br/>giữ newTotalBilling = 0<br/>newNormalBilling = 0]
    
    BranchA --> Step383
    BranchB0 --> Step383
    BranchB1 --> Step383
    B21 --> Step383
    B22 --> Step383
    Step383[3.8.B.3 Insert records<br/>📤 monthly_billing_amounts insert new row<br/>📤 monthly_od_billing_amounts insert new row<br/>📤 monthly_billing_histories insert billing/base_amount/total_repayment_balance]
    Step383 --> Out([to 3.9])
```

---

## 6. Section 3.9 — 少額ユーザ 請求取消

```mermaid
flowchart TD
    Start([3.9 start]) --> AccChk{accountTranferFlg == 0?}
    AccChk -- No --> Skip([skip → 3.12])
    AccChk -- Yes --> Cond1{hasDelay AND NOT hasOverdraft AND NOT hasLost?}
    Cond1 -- Yes --> C1_Diff{sumRepayment - sumRepaid < 1000?<br/>📥 from 3.5 cache}
    C1_Diff -- Yes --> Patch1[PatchRepaymentStatuses02<br/>遅延解除<br/>📤 repayment_statuses applied_flg=0]
    C1_Diff -- No --> Cond2
    Cond1 -- No --> Cond2
    Patch1 --> Cond2{hasOverdraft AND NOT hasLost<br/>AND sumRepayment-sumRepaid<1000<br/>AND total_repayment_balance<1000?<br/>📥 repayment_balances}
    Cond2 -- Yes --> Patch2[PatchRepaymentStatuses02<br/>オーバードラフト解除<br/>📤 repayment_statuses applied_flg=0<br/>📤 repayment_status_histories]
    Cond2 -- No --> End([→ 3.12])
    Patch2 --> End
```

---

## 7. Section 3.12 — 利息計算 (Interest Calc)

```mermaid
flowchart TD
    Start([3.12 entry]) --> G1{stopCalcInterestFlg == 0<br/>AND accountTranferFlg == 0?<br/>📥 repayment_status_controls}
    G1 -- No --> Skip([skip → 3.13])
    G1 -- Yes --> GetRates[GetInterestRateInID ids=1,3<br/>📥 interest_rates<br/>WHERE interest_rate_id IN 1,3]
    GetRates --> RatesErr{rates == nil OR err?}
    RatesErr -- Yes --> Rollback[Wrap 900801000900204<br/>Rollback + retry]
    RatesErr -- No --> Diff[diff = sumRepayment - sumRepaid<br/>📥 from 3.5 cache]
    
    Diff --> Range{Rate range?<br/>📥 interest_rates.lower_limit, upper_limit}
    Range -- lower < diff < upper --> Calc1[rate = interestRateId1.interest_rate<br/>oneDayInterest = rate × diff-sumBill]
    Range -- diff <= lower --> Zero[oneDayInterest = 0]
    Range -- diff >= upper --> Calc3[rate = interestRateId3.interest_rate<br/>oneDayInterest = rate × diff-sumBill]
    
    Calc1 --> IntChk{oneDayInterest >= 1?}
    Calc3 --> IntChk
    Zero --> End
    IntChk -- Yes --> GetTotal[GetTotalInterests<br/>📥 total_interests<br/>WHERE user_id=?]
    GetTotal --> Exists{len totalInterests == 0?}
    Exists -- Yes --> PutTotal[3.12.4.1 PutTotalInterest<br/>📤 total_interests INSERT]
    Exists -- No --> UpdTotal[3.12.5 UpdateTotalInterest<br/>📤 total_interests UPDATE<br/>total_interest = old + oneDayInterest]
    PutTotal --> PutDetail[PutInterestDetails<br/>📤 interest_details INSERT<br/>day, base_amount, interest_rate, interest]
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
    Start([3.13 entry]) --> Calc[preInterestTarget =<br/>repayment_balances.repayment_balance<br/>- sumRepayment - sumRepaid<br/>📥 repayment_balances + from 3.5]
    Calc --> Clamp{oneDayInterest <= 0?}
    Clamp -- Yes --> ZeroInt[oneDayInterest = 0]
    Clamp -- No --> Total
    ZeroInt --> Total[totalRepaymentBalance =<br/>current total_repayment_balance<br/>+ oneDayInterest<br/>+ oneDayDelayInterest]
    Total --> Update[UpdateRepaymentBalance<br/>📤 repayment_balances<br/>total_repayment_balance<br/>interest_target = sumRepayment-sumRepaid<br/>pre_interest_target]
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
        I1[CalcInterest] --> I2[Load user list<br/>📥 S3]
        I2 --> I3[CalcLoop]
    end
    subgraph CalcLoop
        direction TB
        L0[<b>#820</b> GetCommonMinRepaymentRule<br/>📥 common_min_repayment_rules] --> L1[for user in userList]
        L1 --> L2[Begin Tx]
        L2 --> L3[3.4-3.5<br/>📥 repayment_balances<br/>📥 monthly_repayment_balances<br/>📥 direct_debit_payment_terms]
        L3 --> L4[3.6 status gather<br/>📥 repayment_statuses<br/>📥 repayment_status_names<br/>📤 repayment_status_controls]
        L4 --> L5[3.7 status judge<br/>📥 monthly_billing_amounts<br/>📥 monthly_od_billing_amounts<br/>📥 acceleration_notice_dates<br/>📤 repayment_statuses<br/>📤 repayment_status_histories]
        L5 --> L6[3.8 billing calc<br/>📥 <b>#820</b> user_min_repayment_rules<br/>📤 monthly_billing_amounts<br/>📤 monthly_od_billing_amounts<br/>📤 monthly_billing_histories]
        L6 --> L7[3.9 low-amount cancel<br/>📤 repayment_statuses<br/>📤 repayment_status_histories]
        L7 --> L8[3.12 interest calc<br/>📥 interest_rates<br/>📥 total_interests<br/>📤 total_interests<br/>📤 interest_details]
        L8 --> L9[3.13 update balance<br/>📤 repayment_balances]
        L9 --> L10[Commit]
        L10 --> L1
    end
    H4 --> I1
    I3 --> CalcLoop
    L1 -. done .-> Final[notImplementedUsersList → S3 📤]
    Final --> End([return])
```

---

## 12. Điểm thay đổi của Issue #820 (highlighted)

| Nơi | Trước | Sau | Table mới |
|---|---|---|---|
| Đầu `CalcLoop` (line 304-318) | — | Fetch **1 lần** / batch | 📥 `common_min_repayment_rules` |
| Trong for-user (line 1849-1899) | Hard-code `1000` | Priority user > common > error | 📥 `user_min_repayment_rules` |
| `b.2.1` (line 2018-2037) | `newNormalBilling = 1000` | `min(baseBillingAmount, diff)` | — |
| `b.2.2` (line 2039-2050) | `newTotalBilling = newOD`<br/>`newNormalBilling = 0` | **Comment out** (分岐不要) — giữ default 0 | — |

```mermaid
flowchart LR
    subgraph Before[Code cũ]
        BA[hard-code 1000]
        BB[b.2.1: newNormal = 1000]
        BC[b.2.2: newTotal = newOD<br/>when diff < 1000]
    end
    subgraph After[Code mới #820]
        AA[common rule fetch 1 lần<br/>📥 common_min_repayment_rules]
        AB[user rule fetch per user<br/>📥 user_min_repayment_rules<br/>priority user > common > ERROR]
        AC[b.2.1: min baseBilling, diff]
        AD[b.2.2: COMMENTED]
    end
    Before -. #820 .-> After
```

---

## 13. Reference: Full table usage matrix

| Table | Read (📥) | Write (📤) | Section |
|---|---|---|---|
| `users` | user_id, credit_limit (từ S3 / hardcode) | — | 3.3 |
| `repayment_balances` | repayment_balance, total_repayment_balance, interest_target, pre_interest_target | total_repayment_balance, interest_target, pre_interest_target | 3.4, 3.13 |
| `monthly_repayment_balances` | SUM(repayment_balance), SUM(repayed_balance) WHERE date_of_use≤month AND repayed_flg=0 | — | 3.5 |
| `direct_debit_payment_terms` | status=2, apply_start≤today, apply_end>today | — | 3.5-2 |
| `repayment_statuses` | repayment_status_id, applied_flg | applied_flg=0 (cancel), insert new status | 3.6.1, 3.7.1, 3.7.4.A.\*, 3.7.5.1, 3.9.1, 3.9.2 |
| `repayment_status_names` | master data | — | 3.6.2, 3.7.5.2 |
| `repayment_status_controls` | stop_billing_flg, stop_calc_interest_flg, stop_calc_late_charge_flg | các stop_* flags | 3.6.4, 3.7.5.4, 3.8.B.1 |
| `repayment_status_histories` | — | history log khi thay đổi status | 3.7.4.\*, 3.9 |
| `monthly_billing_amounts` | WHERE user_id AND repaid_flg=0, new_total_billing, repaid_billing, billing_month | INSERT new row (new_total_billing, new_normal_billing); PATCH repaid_flg | 3.7.2, 3.8.B.2/3 |
| `monthly_od_billing_amounts` | WHERE user_id AND repaid_flg=0, new_od_billing, repaid_od_billing | INSERT new row; PATCH repaid_flg | 3.7.3, 3.8.B.2/3 |
| `monthly_billing_histories` | — | INSERT: billing, repaid_amount, base_amount, total_repayment_balance, billing_month | 3.8.B.3 |
| `acceleration_notice_dates` | acceleration_month → check 期失通知 | — | 3.7.4.A.2.1.2 |
| `interest_rates` | id=1, id=3 → upper_limit, lower_limit, interest_rate | — | 3.12 |
| `total_interests` | total_interest (check tồn tại) | INSERT / UPDATE total_interest | 3.12.4, 3.12.5 |
| `interest_details` | — | INSERT: day, base_amount, interest_rate, interest | 3.12.4.1 |
| `total_late_charges` | — | (used in late-charge path, not core to #820) | 3.10 |
| `late_charge_detail` | — | (used in late-charge path) | 3.10 |
| `unapplied_repayments` | — | (used in repay-apply path) | 3.11 |
| **`common_min_repayment_rules`** 🆕 | start_month ≤ currentMonth, min_repayment_amount, ORDER start_month DESC LIMIT 1 | — | #820 — line 304-318 |
| **`user_min_repayment_rules`** 🆕 | WHERE user_id AND start_month ≤ currentMonth, min_repayment_amount | — | #820 — line 1849-1868 |
| S3 bucket | UserList.FileName (user list input) | NotImplementedUsersList output | Entry / End |
