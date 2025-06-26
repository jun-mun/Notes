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

## Rpadmin Overview
There are 2 main base classes: 
 - BaseJob
 - BaseEmployerJob

 Listed below are all classes that extend these 2 classes and marks the potential places we need to make changes.... Any one of these jobs can make an update to EmployeeElection, and any place we leave out could be a potential for a bug.

 Files marked as ❌ indicate a file that does not update or insert into EmployeeElections, while a ✔️ indicates that it inserts/updates into EmployeeELections and InputedIncome should be calculated

#### 1. BaseJob
	1. ACAReportingEngineJob ❌ 
	2. ACA_MonthlyEligBatchJob ❌ 
	3. AutoWaiveJob ✔️ 
		calls usp_Batch_AutoWaiveProcessEmployer - insert EmployeeElection
	4. AutoenrollJob  
	5. BaseCobraJob  
	6. CanadianDefaultEnrollmentJob✔️
		Inserts using UESMiddleware EmployeeElectionsDAO
	7. ClosingFTUJob  
	8. CobraCoverageTerminationJob  
	9. CobraDBPProcessJob  
	10. CobraExpirationJob  
	11. CobraSubsidyDisabilityRecalcJob  
	12. DVSSubmissionJob  
	13. DefaultEnrollmentJob  
	14. DepAgeOutJob ❌ 
	15. DepTermJob ✔️
	16. DependentAgeOfAttainmentJob ❌  
	17. ECAutoAllocationJob 
	18. EligCacheJob
	19. EmployeeAgingJob 
	20. EmployeeCoverageOfferedJob
	21. EmployeeRatesELigCacheJob
	22. EmployerMandateJob
	23. EmployerMandateJobScheduler
	24. EngagementVerificationJob
	25. EoiJob
	26. FulltimeStudentAuditJob
	27. FutureAutowaiveJob
	28. GroupStructureJOb
	29. GroupStructureJobScheduler
	30. HPCCUDPReconciliationJob
	31. INGEoiJob
	32. LOAFutureElectionTerminationsJob
	33. LOSJob
	34. LOSResetJob
	35. MissingDepSSNNoticesJob
	36. PlanEligAgeJob
	37. RateRecalcJob✔️
	38. RenewalEngineJob
	39. RenewalJob✔️
	40. ReportJob
	41. SpouseAgingJob
	42. UDPPlanReconciliationJob
	43. UDPRequestReconciliationJob
	44. VaccineVerificationJob
	45. VaccineVerificationOngoingJob
	46. WellnessCreditJob
	47. WellnessProgramJob

#### 2. Implementations of BaseEmployerJob
	1. ACA_MonthlyEligBatchERJob
	2. AutoEnrollERJob
	3. AutoWaiveERJob️️✔️
	4. BaseDefaultEnrollmentERJobStep
	5. CanadianDefaultEnrollmentERJob
	6. CobraCoverageTerminationERJob
	7. CobraExpirationERJob
	8. CobraSubsidyDisabilityRecalcERJob
	9. DefaultEnrollmentERJob
	10. DefaultEnrollmentERJobEmployerContribution
	11. DefaultEnrollmentERJobWellnessPrograms
	12. DepAgeOutERJob
	13. DepTermERJob✔️
	14. DependentAgeOfAttainmentERJob
	15. ECAutoAllocationERJob
	16. EligCacheERJob
	17. EmployeeAgingERJob
	18. EmployeeRatesEligCacheERJob
	19. EmployerMandateERJob
	20. EngagementVerificationERJob 
	21. EOiERJob
	22. FulltimeStudentAuditERJob
	23. FutureAutowaiveERJob
	24. GroupStructureERJob
	25. HPPCUDPReconciliationERJob
	25. LOAFutureElectionTerminationERJob
	26. LOSERJob
	27. LOSResetERJob
	28. MissingDepSSNNoticesERJob
	29. PlanEligAgeGainERJob
	30. PlanEligAgeLossERJob
	31. RenewalERJob✔️
	32. SpouseAgingERJob
	33. UDPPlanReconciliationERJob
	34. UDPRequestReconciliationERJob
	35. VaccineVerificationERJob
	36. WellnessCreditERJob
	37. WellnessProgramERJob
	38. 


	

 
### Rate Recalculation
1. Identify where rate recalculations are performed.
2. Find any updates or references to EmployeeElections
3. Look for updates to `ImputedIncome` column, we need to make sure they get updated to EmployeeElection.


The RateRecalculateionJob overrides the BaseJob#executeJob method and calls the **sp_rate_recalc_batch** stored procedure. 
```java
@Override
	public void executeJob(final JobExecutionContext context, final JobRunDAO jobRunDAO, final JobRunBean jobRunBean, final StatusLogBean statusLogBean) {
		statusLogBean.logJobProgress("Rate Recalculation", 1, 0, jobRunBean.getStart(), true, "Recalculation");
		try {
            // call to sp_rate_recalc_batch
			ConnectionFactory.getInstance().getConnection().createStatement().execute("EXECUTE sp_rate_recalc_batch");
		} catch (Exception ex) {
			statusLogBean.addToRunLog("Error executing the Rate Recalculation stored procedure");
			ex.printStackTrace();
		}
	}
```
#### **sp_rate_recalc_batch**
The stored procedure is broken up by if statements depending on the **@rcd_update_type**. But for almost all update types,
there are some updates and insertion queries into the EmployeeElection table. 

```sql
while (select count(*) from #tmp_ui_calcs) > 0 begin

            set @calc_id = (select top 1 rcr_id from #tmp_ui_calcs order by rcr_id)

            if object_id('tempdb..#tmp_ui_orig_election') is not null drop table #tmp_ui_orig_election
            --Store off the original employee election
            select el.*
            into #tmp_ui_orig_election
            from #tmp_ui_calcs inner join employeeelections el on elect_id = rcr_electid
            where rcr_id = @calc_id

            -- Update the exisiting election record
            update employeeelections set elect_premium = rcr_elect_premium, elect_ee_prem = rcr_elect_ee_prem, elect_er_prem = rcr_elect_er_prem, elect_ee_payfreq_prem = rcr_elect_ee_payfreq_prem, elect_er_payfreq_prem = rcr_elect_er_payfreq_prem, elect_ee_pretax_payfreq_prem = isnull(rcr_elect_ee_pretax_payfreq_prem,0), elect_ee_pretax_prem = isnull(rcr_elect_ee_pretax_prem,0), elect_tax_status = rcr_elect_tax_status,elect_amount = case when plan_type = 3 then rcr_elect_amount else el.elect_amount end, elect_coverage_level = case when plan_type = 3 then rcr_elect_amount else el.elect_coverage_level end, elect_tran_date = getdate(),
            elect_excess_applied_prem = rcr_elect_excess_applied_prem, elect_excess_applied_payfreq_prem = rcr_elect_excess_applied_payfreq_prem , elect_excess_er_payfreq_prem = rcr_elect_excess_earned_payfreq_prem
            from #tmp_ui_orig_election te inner join employeeelections el on el.elect_id = te.elect_id and el.elect_eeid = te.elect_eeid
            inner join #tmp_ui_calcs tc on tc.rcr_electid = te.elect_id
            inner join plans on plan_id = el.elect_planid

            set @update_orig_el_cnt = @update_orig_el_cnt + @@rowcount
```

### Renewal
1. Identify where renewals are processed.
2. Add logic to calculate imputed income during renewal.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.


#### RenewalWorker
**There are around 7 renewal worker classes that extend RenewalWorker, RenewalERBatch eventually gets a list of renewal workers to go through and calls the RenewalWorkder#renew**. We could implement a method in RenewalWorker and call it at the end of renew to make sure we cover all the renewal flows. These call the EmployeeElectionServer#saveElection (UESMiddleware)

** I did not chekck all the implementations of RenewarWorker, but the 3 I checked all used EmployeeElectionDAO (UESMiddleware) for update and insertions

### Dependent Age 
1. Identify where dependent age-out events are handled.
2. Add logic to calculate imputed income when a dependent ages out.
3. Update the `ImputedIncome` column with the calculated value.
4. Log the change in the audit trail.

#### 1. Dependent Age of Attainment ER
Looks like this just looks through the student list and sends notifications

#### 2. Dependent Term
Creates a DepTermERJob instance and calls the DepTermERJob#execute.

This job eventually calls USP_DenyDependentProcess and proc_usp_RemoveDependentFromCoverage, which has an insert into the EmployeeElection table

```
```
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



---
### List of Insert/Update to EmployeeElection in rpadmin
1. sp_rate_recalc_batch (rate recalc)
2. Renewal (EmployeeELectionDAO)
3. USP_DenyDependentProcess (Dependent Term)
4. proc_usp_RemoveDependentFromCoverage (Dependent Term)
