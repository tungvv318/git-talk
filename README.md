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

---

# Version 2 — Flow + Business Logic

Phần này bổ sung **WHY** (nghiệp vụ đằng sau) cho từng step. Version 1 ở trên trả lời **WHAT/HOW** (kỹ thuật).

## V2.1 — Bối cảnh nghiệp vụ tổng quan

Batch `bt-intcalc` là **金利計算バッチ** (Interest Calculation Batch), chạy **mỗi ngày** để:

1. **Tính lãi hàng ngày** cho từng khoản vay (sumRepaymentBalance × interest_rate).
2. **Sinh request billing** hàng tháng cho user (số tiền tối thiểu phải trả).
3. **Cập nhật trạng thái nợ**: phát hiện trễ hạn (delay), vỡ nợ (lost/期失), vượt hạn mức (overdraft).
4. **Xử lý miễn trừ**: user đăng ký 口座振替 (auto-debit), user nợ nhỏ (< ¥1000) được cancel billing.

**Stakeholder & constraint:**
- Legal/Compliance: phải log đầy đủ lịch sử thay đổi status (`repayment_status_histories`).
- Ops: nếu 1 user fail, **không** được block toàn batch — chỉ mark user đó vào `notImplementedUsersList` để retry thủ công.
- Performance: có thể chạy với **1000+ users/đêm** → tối ưu query (#820 split common rule fetch).

```mermaid
flowchart LR
    subgraph Daily[Mỗi ngày 深夜バッチ]
        D1[Tính lãi 1 ngày<br/>per user]
        D2[Update tổng nợ<br/>repayment_balances]
    end
    subgraph Monthly[Đầu tháng]
        M1[Sinh billing amount<br/>= min baseRule, outstanding]
        M2[User thanh toán qua<br/>口座振替 / 手動]
    end
    subgraph Exception[Khi có bất thường]
        E1[Phát hiện trễ hạn<br/>→ status 002 遅延]
        E2[Vượt hạn mức<br/>→ status 001 OD]
        E3[Vỡ nợ → 003 期失<br/>acceleration notice]
    end
    Daily --> Monthly
    Daily --> Exception
```

---

## V2.2 — Business logic cho từng section

### 3.4 — GetRepaymentBalance (với `FOR UPDATE`)

```mermaid
flowchart LR
    A[GetRepaymentBalance FOR UPDATE] --> B[Lock row của user<br/>trong transaction]
    B --> C[Đảm bảo không có<br/>batch/API khác ghi song song]
```

**Business reason:**
- `repayment_balances` là **source of truth** cho tổng dư nợ.
- Nếu API user-facing cập nhật cùng lúc → race condition → tính lãi sai.
- `SELECT FOR UPDATE` lock row cho đến Commit/Rollback → atomic per-user.

---

### 3.5 — SUM monthly_repayment_balances với `date_of_use <= month-2`

**Business reason:**
- `month` truyền vào = `time.Now().Month() - 2` (2 tháng trước) — KHÔNG phải tháng hiện tại.
- Nghiệp vụ: **利息は利用確定後に課す** (lãi chỉ tính sau khi giao dịch đã confirmed). Thẻ tín dụng có 請求確定期間 khoảng 2 tháng → chỉ tính lãi trên phần **đã confirmed**.
- Do đó `date_of_use <= '202602'` (khi chạy 2026-04) nghĩa là: "tính lãi trên các giao dịch đã confirmed đến hết tháng 2".

```mermaid
flowchart LR
    T1[TX tháng 4] -.chưa confirm.-> NoInterest[KHÔNG tính lãi]
    T2[TX tháng 3] -.đang confirm.-> NoInterest
    T3[TX tháng 2 hoặc trước] --confirmed--> Interest[TÍNH LÃI]
```

---

### 3.5-2 — direct_debit_payment_terms → `accountTranferFlg`

**Business reason:**
- User đăng ký 口座振替 (auto-debit từ tài khoản ngân hàng) được đối xử **khác** trong mọi bước downstream:
  - **3.8**: không sinh OD billing (tiền sẽ bị debit tự động, không cần notify)
  - **3.9**: cho phép cancel billing nhỏ (vì auto-debit sẽ tự refund qua clearing)
  - **3.12**: **không** tính lãi bằng batch (đã tính riêng trong direct-debit flow)
- Nếu user có thay đổi kế hoạch auto-debit, cột `status='2'` = active contract.

---

### 3.6 – 3.7 — Status judgment (delay / lost / overdraft)

```mermaid
flowchart TD
    Check[Fetch repayment_statuses<br/>current flags] --> Compute[Compute:<br/>hasDelay, hasLost, hasOD]
    Compute --> Branch{Condition satisfied?}
    Branch -- Payment cleared --> B_Rel[Release status:<br/>applied_flg = 0<br/>📤 history log]
    Branch -- Still overdue --> B_New[Insert new status:<br/>applied_flg = 1<br/>📤 history log]
    
    subgraph BusinessRules[Business rules]
        R1[遅延 002: quá hạn 1 tháng]
        R2[OD 001: vượt credit_limit]
        R3[期失 003: quá hạn 3 tháng<br/>sau khi gửi 期失通知]
        R4[委託 013: đưa sang đòi nợ]
        R5[停止 014: ngừng cấp tín dụng]
    end
```

**Business reason:**
- Status **lifecycle**: 正常 → 遅延 (quá 1 tháng) → 期失 (quá 3 tháng + notice sent) → 委託/停止.
- Mỗi lần chuyển status phải ghi `repayment_status_histories` (legal requirement: truy xuất lịch sử nợ).
- Nếu user trả hết → tự động `applied_flg=0` (release status) thay vì DELETE → giữ lịch sử.

---

### 3.8 — Billing Judgment (phần được modify #820)

```mermaid
flowchart TD
    Start[Start 3.8] --> BR[baseBillingAmount<br/>= min payment per month<br/>theo hợp đồng]
    BR --> Business1[<b>Business rule</b>:<br/>Mỗi tháng user phải trả ít nhất<br/>baseBillingAmount]
    Business1 --> Stop{stop_billing_flg?}
    Stop -- Admin hold --> Skip[Không sinh billing<br/>Marking repaid để không tính thêm]
    Stop -- Normal --> AccTxf{auto-debit?}
    AccTxf -- Yes --> B0[Không OD billing<br/>toàn bộ auto-debit toàn outstanding]
    AccTxf -- No --> Lost{期失?}
    Lost -- Yes --> B1[Full outstanding<br/>đòi toàn bộ vì đã vỡ nợ]
    Lost -- No --> B2[Normal case:<br/>Đòi min baseBilling, outstanding]
    
    B2 --> Before[<b>Trước #820:</b> hardcode 1000]
    B2 --> After[<b>#820:</b> dynamic từ DB<br/>user-specific hoặc global]
```

**Business reason các nhánh:**

| Nhánh | Scenario | Business logic |
|---|---|---|
| **a) stop_billing_flg=1** | Admin tạm dừng billing (vd: bug, dispute) | Không sinh request → user không thấy invoice mới. Đánh `repaid_flg=1` để loop sau không revisit. |
| **b.0) account_transfer_flg=1** | User đã đăng ký 口座振替 | OD không cần vì auto-debit sẽ kéo tiền về. `newTotalBilling = total outstanding` (debit full amount). |
| **b.1) hasLostStatus** | User đã 期失 (vỡ nợ, quá 3 tháng) | Đòi toàn bộ dư nợ còn lại — không có khái niệm "trả tối thiểu" nữa. |
| **b.2) Normal case** | User thường, chưa vỡ nợ, không auto-debit | Đòi `min(baseBillingAmount, diff)` — đảm bảo user trả tối thiểu nhưng không quá dư nợ. |

**Tại sao #820 phải dynamic baseBillingAmount?**

```mermaid
flowchart LR
    Before[Trước: hardcode 1000] --> Problem1[Không config được<br/>theo user hoặc theo thời gian]
    Problem1 --> Problem2[Muốn tăng/giảm<br/>phải deploy code]
    Before --> Before2[Ảnh hưởng tới chính sách:<br/>VD regulation thay đổi<br/>base amount]
    After[Sau: DB config] --> Benefit1[Thay giá trị không cần deploy]
    After --> Benefit2[User VIP có thể override<br/>user_min_repayment_rules]
    After --> Benefit3[Dùng start_month<br/>để schedule thay đổi trước]
```

**Tại sao b.2.2 bị comment (分岐不要)?**

- b.2.2 cũ: khi `diff < 1000` → `newTotalBilling = newODBilling`, `newNormalBilling = 0`.
- Nghiệp vụ sau review: scenario này **hiếm xảy ra** với logic upstream (nếu diff < 1000, thường đã được cancel ở 3.9). Giữ default `0` thay vì chạy nhánh riêng → đơn giản hoá.
- ⚠ Vẫn cần PM confirm: nếu có user OD nhưng diff < baseBilling, code mới sẽ **không** bill OD phần đó nữa.

---

### 3.9 — Low-amount billing cancel (<¥1000)

**Business reason:**
- Khi dư nợ < ¥1000 và user ở trạng thái 遅延/OD, batch tự động release status.
- Lý do: chi phí đòi ¥1000 > giá trị nợ. Rounding policy.
- CHỈ áp dụng cho **auto-debit user** (`accountTranferFlg=0` ngược nghĩa — xem lại code: actually condition là `accountTranferFlg == 0` tức **không auto-debit**).

---

### 3.12 — Interest Calculation

```mermaid
flowchart TD
    Diff[diff = sumRepayment - sumRepaid<br/>= outstanding principal] --> Range{diff vs limits?}
    Range -- diff <= 下限 --> NoInt[oneDayInterest = 0<br/>nợ quá nhỏ không tính lãi]
    Range -- 下限 < diff < 上限 --> Tier1[Rate 1 áp dụng<br/>vd 15% APR]
    Range -- diff >= 上限 --> Tier3[Rate 3 áp dụng<br/>vd 18% APR - high tier]
    
    Tier1 --> Formula[oneDayInterest =<br/>rate × outstanding × 1/365]
    Tier3 --> Formula
    
    Formula --> Accum{total_interests exists?}
    Accum -- No --> Insert[INSERT new row<br/>lần đầu trong tháng]
    Accum -- Yes --> Update[UPDATE:<br/>total += today's interest]
    
    Insert --> Detail[INSERT interest_details<br/>ghi lại lãi mỗi ngày<br/>cho audit/legal]
    Update --> Detail
```

**Business reason:**
- **Tiered rate**: Japan Interest Rate Act — nợ lớn có lãi suất cao hơn theo tầng. `interest_rates` table config 2 tier (id=1 thấp, id=3 cao).
- **Per-day calc**: `rate × diff / 365` (lãi đơn ngày) → cộng dồn vào `total_interests` hàng tháng. Cuối tháng, total_interest sẽ thành 1 billing entry cho tháng sau.
- **`interest_details` audit log**: mỗi ngày 1 row với base_amount + rate + interest → required for dispute handling.
- **Guard điều kiện `stopCalcInterestFlg=0 && accountTranferFlg=0`**: admin có thể ngừng lãi (customer service), auto-debit user đã tính lãi ở pipeline khác.

---

### 3.13 — Update repayment_balances

**Business reason:**
- Sau khi cộng lãi ngày hôm nay, `total_repayment_balance` = tổng nợ mới (principal + interest + late_charge).
- `interest_target` = principal chưa trả (base để tính lãi ngày mai).
- `pre_interest_target` = snapshot của interest_target trước khi update (for delta calc).

---

## V2.3 — Transaction boundary & data consistency

```mermaid
flowchart TD
    Begin[BEGIN TRANSACTION per user] --> L1[SELECT repayment_balances FOR UPDATE<br/>🔒 LOCK user row]
    L1 --> Calc[...3.5 → 3.13...]
    Calc --> Result{Any error?}
    Result -- Yes --> Rollback[ROLLBACK<br/>→ user data giữ nguyên<br/>→ marked notImpl để retry]
    Result -- No --> Commit[COMMIT<br/>→ atomic update mọi table]
    
    subgraph Consistency[Tại sao atomic quan trọng?]
        C1[Ghi total_interests<br/>+ interest_details<br/>+ repayment_balances<br/>+ monthly_billing_*]
        C2[Nếu commit partial<br/>sẽ sai tổng số]
    end
```

**Business reason:**
- **Atomicity requirement (legal)**: `total_interests` ghi "đã tính lãi" nhưng `repayment_balances.total_repayment_balance` chưa cộng → user bị charge 2 lần khi batch chạy lại.
- **Row-level lock vs table-level**: dùng `FOR UPDATE` trên `repayment_balances` cho từng user → các user khác vẫn chạy parallel.
- **Retry tại user granularity**: 1 user fail không block N-1 users khác.

---

## V2.4 — Ảnh hưởng #820 lên business logic

```mermaid
flowchart TB
    subgraph OldBehavior[Trước #820]
        O1[1000 yen<br/>hardcode cho mọi user]
        O2[Không phân biệt<br/>regular vs VIP]
        O3[Thay đổi phải<br/>deploy code]
    end
    
    subgraph NewBehavior[Sau #820]
        N1[Giá trị config<br/>ở DB]
        N2[User rule override<br/>common rule]
        N3[Thay đổi qua<br/>INSERT/UPDATE SQL]
        N4[Schedule thay đổi<br/>bằng start_month tương lai]
    end
    
    subgraph BusinessValue[Business value]
        B1[Regulation compliance:<br/>Nhanh chóng điều chỉnh<br/>khi luật đổi]
        B2[Customer segmentation:<br/>VIP trả min thấp hơn]
        B3[A/B test:<br/>Thử nghiệm mức base<br/>không cần release]
        B4[Audit: <br/>start_month + pgm_id<br/>→ truy xuất ai/khi nào đổi]
    end
    
    OldBehavior -. #820 .-> NewBehavior
    NewBehavior --> BusinessValue
```

**Tác động tới test:**
- **TC4 (no rule → error)**: đây là **safety guard** — nếu ops quên INSERT rule sau migration, batch fail an toàn thay vì dùng giá trị sai.
- **TC6 (latest rule)**: support schedule — có thể insert rule cho `start_month` tương lai, sẽ tự active khi tới tháng đó.
- **TC3 (rollback)**: đảm bảo không có partial write → user không bị sai data khi config thiếu.

---

## V2.5 — Failure modes & mitigation (business perspective)

| Failure | Tác động business | Mitigation trong code |
|---|---|---|
| DB connection mất giữa chừng | User bị sai billing | Rollback toàn transaction, retry 3 lần (handler.go:94) |
| `common_min_repayment_rules` bị xóa nhầm | Batch fail toàn bộ (blocking) | Error `900801000900204` + retry. Ops phải restore rule. |
| User rule mâu thuẫn common rule | User bị billing sai | Priority hierarchy: user > common (explicit logic tại b.2) |
| Interest rate changed mid-batch | 1 phần user dùng rate cũ, 1 phần rate mới | Batch fetch `interest_rates` mỗi user (consistent trong tx) |
| User có 期失 nhưng cũng OD | Confusion trong xử lý | Priority: Lost (b.1) > OD check → luôn bill full |
| Duplicate batch run (same day) | Double billing / double interest | `total_interests` check existence → UPDATE thay vì INSERT. `monthly_billing_amounts` có unique (user_id, billing_month). |

