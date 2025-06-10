# 🧾 Technical Analysis Doc: Imputed Income Update Events

## 📌 Objective
To identify where imputed incomes are calculated and add a step to store the imputed income value in a new column when any of the following events occur:
- Rate Recalculation  
- Renewal  
- Dependent Age Out  
- Employee Aging (Life/Disability)  
- Spouse Aging (Life)

---

## ❓ Questions:
- Any other rate recalculations? How does it work with batch vs user updates

---
##  ✅ TODO section
 
### Rate Recalculation
1. Identify where rate recalculations are performed.
2. Add logic to calculate imputed income based on new rates.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.

### Renewal
1. Identify where renewals are processed.
2. Add logic to calculate imputed income during renewal.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.

### Dependent Age Out
1. Identify where dependent age-out events are handled.
2. Add logic to calculate imputed income when a dependent ages out.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.

### Employee Aging (Life/Disability)
1. Identify where employee aging events are processed.
2. Add logic to calculate imputed income based on new age-based rates.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.

### Spouse Aging (Life)
1. Identify where spouse aging events are handled.
2. Add logic to calculate imputed income based on new age-based rates.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.

---



## 🧩 1. Event Definitions

| **Event**                  | **Trigger Condition**                                                                 |
|---------------------------|----------------------------------------------------------------------------------------|
| Rate Recalculation         | Change in benefit rate due to salary update, plan change, or policy adjustment       |
| Renewal                   | Annual or periodic plan renewal                                                       |
| Dependent Age Out         | Dependent reaches age limit for coverage (e.g., 26 years old)                         |
| Employee Aging            | Employee reaches age bracket affecting Life/Disability rates                          |
| Spouse Aging              | Spouse reaches age bracket affecting Life insurance rates                             |

---

## 🧮 2. Imputed Income Calculation Logic

| **Input Fields**                  | **Description**                                                                 |
|----------------------------------|---------------------------------------------------------------------------------|
| Coverage Amount                  | Total coverage provided (e.g., \$100,000)                                       |
| IRS Threshold                    | \$50,000 (for group-term life insurance)                                        |
| Age-Based Rate Table             | Monthly cost per \$1,000 of coverage based on age bracket                       |
| Taxable Amount                   | (Coverage - \$50,000) / \$1,000 × Rate                                          |
| Frequency                        | Monthly or per pay period                                                       |

---

## 🛠️ 3. System Requirements

- **New Column**: `ImputedIncome` (Decimal, currency format)
- **Trigger Points**: System must recalculate and update `ImputedIncome` when any of the 5 events occur.
- **Audit Trail**: Log changes with timestamp, event type, and previous value.
- **Validation**: Ensure age and coverage data are current before calculation.

---

## 🔄 4. Workflow Example: Employee Aging

1. **Trigger**: Employee turns 50.
2. **System Action**:
   - Fetch new rate from age-based table.
   - Recalculate imputed income.
   - Update `ImputedIncome` column.
   - Log change in audit table.

---

## 📊 5. Reporting & Monitoring

- **Monthly Report**: List of all records with updated `ImputedIncome`
- **Exception Report**: Records missing age or coverage data
- **Audit Log**: Event type, old value, new value, timestamp, user/system ID
