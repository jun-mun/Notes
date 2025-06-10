# Analysis of the `GetEventsWithActionsByEeId` Stored Procedure

This stored procedure is used to retrieve events that need to be processed for a specified employee in the Event Engine. It's primarily focused on determining actions that should be taken based on events like service area changes, terminations, and leave of absence (LOA) events.

## Section 1: Header and Documentation (Lines 1-59)
```sql
-- filepath: c:\projects\CBA_cba-db-sources\ues\Proc_GetEventsWithActionsByEeId.sql
USE [UES];
GO
SET QUOTED_IDENTIFIER ON;
GO

-- =====================================================================================================================
-- If the procedure does not exist then create a stub
-- =====================================================================================================================
CREATE OR ALTER PROCEDURE dbo.GetEventsWithActionsByEeId 
    @eeId INT
AS
BEGIN
	SET ANSI_NULLS ON;
    SET NOCOUNT ON;
    SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

/*
Filename:  Proc_GetEventsWithActionsByEeId.sql
This procedure returns all events that need to be processed to the EventEngine. The service area loss/gain events that were created in the previous
stored procedure will then be used to create the EventEnrollmentAction records which are used to determine if the employee can elect a new plan based
on the life event rules.
Input params:
    @eeId int - employee id
Output params:

Return values:

Update Log
Date        Author          Description
...
*/
```

This section establishes the procedure header with a single parameter `@eeId` (employee ID), sets SQL environment settings, and includes comprehensive documentation about the purpose and change history.

## Section 2: Variable Declarations (Lines 60-142)
```sql
    DECLARE @erid INT
    DECLARE @sagain_evtdt DATETIME
    DECLARE @sellists TABLE (eliggrp_id INT, sellist_id INT, bndlplan_id INT, plan_type INT, sam_id INT, plan_id INT, plan_subtype VARCHAR(50))
    DECLARE @sellists2 TABLE (eliggrp_id INT, sellist_id INT, bndlplan_id INT)
    DECLARE @sagSellists TABLE (sag_sellistid INT, inOld INT, inNew INT)
    DECLARE @activeBndlplans table (bndlplanId INT, sellistId INT)
    DECLARE @sellistId INT,@electId INT, @loaSetupExists BIT
	DECLARE @lifeAfterCobraEventExists BIT = 0;
	DECLARE @employeeTerminatedStatus BIT = 0;
    DECLARE @sellistsEAP TABLE (eliggrp_id INT, sellist_id INT, bndlplan_id INT, plan_type INT, sam_id INT, plan_id INT)
	DECLARE @elevatedDepedentWithAdminEvent BIT = 0;
	DECLARE @sellistsElevatedDependents TABLE (ee_id INT, eliggrp_id INT, sellist_id INT, bndlplan_id INT, plan_type INT, sam_id INT, plan_id INT)    
	DECLARE @activeSellistsAdminAfterCobraEvent TABLE (ee_id INT, eliggrp_id INT, sellist_id INT, bndlplan_id INT, plan_type INT, sam_id INT, plan_id INT, plan_subtype VARCHAR(50), planyear CHAR(4))    
	DECLARE @cobraSellistsAdminAfterCobraEvent TABLE (ee_id INT, eliggrp_id INT, sellist_id INT, bndlplan_id INT, plan_type INT, sam_id INT, plan_id INT, plan_subtype VARCHAR(50), planyear CHAR(4))    
	DECLARE @adminAfterCobraEvent BIT = 0;
	DECLARE @ee_erid INT;
	DECLARE @eed_date_of_leave DATETIME;
	DECLARE @curr_status INT;
	DECLARE @curr_reason INT;
	DECLARE @curr_reason_code VARCHAR(255);
	DECLARE @prev_status INT;
	DECLARE @prev_reason INT;
	DECLARE @prev_reason_code VARCHAR(255);
	DECLARE @eed_active_status_date DATETIME;

    DECLARE @unionTable1 TABLE
    (
             evtId INT, evtEeId INT, evtErId INT, evtType INT, evtReasonId INT, evtSource INT, evtEligGrpId INT, evtDate DATETIME,
             evtDetDate DATETIME, evtCause INT,eligGrpPlanYrEnd DATETIME, procAction INT, evtActElectId INT, evtActAction INT,
             evtActElectStatus INT, evtActEffDateTiming INT, evtActActionWindow INT, evtActLifeStatusChangeEffDateWaitPeriod INT, evtActBndlPlanId INT,
             evtActRuleId INT, evtActSelListId INT, evtEffDate DATETIME, causedEvtId INT, causedEvtType INT, causedEvtReasonId INT,
             causedEvtSource INT, causedEvtEligGrpId INT, causedEvtDate DATETIME, causedEvtDetDate DATETIME, causedEvtAction INT,
             causedEvtEffDateTiming INT, causedEvtWindow INT, causedEvtLSChangeEffDateWaitingPeriod INT, causedEvtRuleId INT,
             causedEvtEffDate DATETIME, evtFirstOfMonthRule INT,electTranDate DATETIME
    );
```

This section declares all the variables that will be used throughout the procedure, including:
- Simple variables to hold IDs, dates, and status values
- Table variables to store temporary result sets for selection lists and service area data
- A primary table variable `@unionTable1` that will accumulate all the events and actions

## Section 3: Initial Status Checks and Data Gathering (Lines 143-199)
```sql
	SELECT @lifeAfterCobraEventExists = ues.cobra.udf_checkForOpenLifeEventAfterClosedInitialCobraEvent(@eeId);
	IF EXISTS (SELECT 1 FROM ues.dbo.employee WHERE ee_id = @eeId AND eed_employment_status = 'Terminated')
		BEGIN
			SET @employeeTerminatedStatus = 1;
		END
		
	SELECT @adminAfterCobraEvent = cobra.udf_checkForAdminAfterCobraEvent (@eeId);
			
	SELECT @elevatedDepedentWithAdminEvent = cobra.udf_checkForElevatedDependentWithAdminEvent (@eeId);	

	-- Populate loa reason variables
	SELECT @ee_erid = ee.ee_erid, @eed_date_of_leave = ee.eed_date_of_leave, @curr_status = ee.curr_status, @curr_reason = ee.curr_reason, 
		   @curr_reason_code = ee.curr_reason_code, @prev_status = ee.prev_status, @prev_reason = ee.prev_reason, @prev_reason_code = ee.prev_reason_code,
           @eed_active_status_date = ee.eed_active_status_date
	FROM ues.dbo.udf_GetLOAReasonByEE_ID(@eeId) ee;

    -- check if new LOA setup for employer in RI
    IF EXISTS 
         (SELECT 1
          FROM ues.dbo.LOAEvents loa 
		  WHERE loa.loae_erid = @ee_erid
				AND EXISTS 
					( SELECT 1 
					  FROM ues.dbo.LOAEventsCache loaecache 
					  WHERE loaecache.loaecache_loaeid = loa.loae_id
							AND loaecache.loaecache_loaescid = @curr_status AND (loaecache.loaecache_loarcid = @curr_reason OR @curr_reason_code = 'All')
							AND loaecache.loaecache_from_loaescid = @prev_status AND (loaecache.loaecache_from_loarcid = @prev_reason OR @prev_reason_code = 'All')))
	BEGIN
		SET @loaSetupExists = 1;
	END
	ELSE
	BEGIN
		SET @loaSetupExists = 0;
	END;
```

This section performs initial checks on the employee's status:
- Checks for life events after COBRA events
- Determines if the employee is terminated
- Checks for admin events after COBRA
- Checks for elevated dependents with admin events
- Retrieves Leave of Absence (LOA) information
- <mark>Determines if LOA setup exists in the system</mark>

## Section 4: Data Population - Selection Lists (Lines 200-297)
```sql
    SET @sagain_evtdt = NULL

    SELECT @erid = ee_erid FROM ues.dbo.employee WHERE ee_id = @eeid

    SELECT @sagain_evtdt = detevt_evtdate FROM ues.dbo.DetectEvents WHERE detevt_eeid = @eeid AND detevt_type = 38 AND detevt_status = 1
	IF (@adminAfterCobraEvent = 1)
		BEGIN
			IF (@employeeTerminatedStatus <> 1)
				BEGIN
		    		INSERT INTO @activeSellistsAdminAfterCobraEvent
					SELECT @eeId, eeg.eliggrp_id, sl.sellist_id, bp.bndlplan_id AS bndlplan_id, p.plan_type, samm.samm_samid, p.plan_id, p.plan_subtype, eg.eliggrp_planyear
					FROM ues.dbo.Employee e
					INNER JOIN ues.dbo.EE_EligGroups eeg ON eeg.ee_id = e.ee_id AND e.ee_id = @eeId
					INNER JOIN ues.dbo.EligGroups eg ON eg.eliggrp_id = eeg.eliggrp_id
					INNER JOIN ues.dbo.SelectionLists sl ON eeg.eliggrp_id = sl.sellist_eliggrpid
					INNER JOIN ues.dbo.Bundles b ON b.bndl_sellistid = sl.sellist_id
					INNER JOIN ues.dbo.BundlePlan bp ON b.bndl_id = bp.bndlplan_bndlid
					INNER JOIN ues.dbo.Plans p ON p.plan_id = bp.bndlplan_planid
					LEFT OUTER JOIN ues.dbo.ServiceAreaMasterMapping samm ON samm.samm_planid = p.plan_id
					WHERE bp.bndlplan_effto_date > getdate()
					GROUP BY eeg.eliggrp_id, sl.sellist_id, samm.samm_samid, bp.bndlplan_id, p.plan_type, p.plan_id, p.plan_subtype, eg.eliggrp_planyear		    		
				END
			
		    INSERT INTO @cobraSellistsAdminAfterCobraEvent		
			SELECT @eeId, ceeeg.ceeeg_eliggrpid, sl.sellist_id, bp.bndlplan_id AS bndlplan_id, p.plan_type, samm.samm_samid, p.plan_id, p.plan_subtype, eg.eliggrp_planyear
			FROM ues.dbo.Employee e
			INNER JOIN ues.cobra.COBRAEEEligGroups ceeeg ON ceeeg.ceeeg_eeid = e.ee_id AND e.ee_id = @eeId
			INNER JOIN ues.dbo.EligGroups eg ON eg.eliggrp_id = ceeeg.ceeeg_eliggrpid
			INNER JOIN ues.dbo.SelectionLists sl ON ceeeg.ceeeg_eliggrpid = sl.sellist_eliggrpid
			INNER JOIN ues.dbo.Bundles b ON b.bndl_sellistid = sl.sellist_id
			INNER JOIN ues.dbo.BundlePlan bp ON b.bndl_id = bp.bndlplan_bndlid
			INNER JOIN ues.dbo.Plans p ON p.plan_id = bp.bndlplan_planid
			LEFT OUTER JOIN ues.dbo.ServiceAreaMasterMapping samm ON samm.samm_planid = p.plan_id
			WHERE bp.bndlplan_effto_date > getdate()
			GROUP BY ceeeg.ceeeg_eliggrpid, sl.sellist_id, samm.samm_samid, bp.bndlplan_id, p.plan_type, p.plan_id, p.plan_subtype, eg.eliggrp_planyear
            
            /* Additional code for handling special cases */
        END
    ELSE IF (@elevatedDepedentWithAdminEvent = 1)
        BEGIN
            /* Code for elevated dependents with admin event */
        END
    IF (@elevatedDepedentWithAdminEvent <> 1 AND @adminAfterCobraEvent <> 1)
        BEGIN    
            /* Code for standard selection lists */
        END
```

This section populates selection lists based on the employee's situation:
- Retrieves the employer ID
- Checks for service area gain events
- Handles different scenarios for selection lists:
  - Admin events after COBRA
  - Elevated dependents with admin events
  - Standard selection lists for regular employees

## Section 5: Additional Data Population (Lines 299-359)
```sql
    /* Life Event after initial cobra ALWAYS GET EAP plans */
	IF(@lifeAfterCobraEventExists = 1 AND @employeeTerminatedStatus = 1)
		BEGIN
			INSERT INTO @sellistsEAP (eliggrp_id, sellist_id, bndlplan_id, plan_type, sam_id, plan_id)
			SELECT eg.eliggrp_id, sl.sellist_id, bp.bndlplan_id, p.plan_type, samm.samm_samid, p.plan_id
			FROM ues.dbo.BundlePlan bp
				INNER JOIN ues.dbo.Plans p ON p.plan_id = bp.bndlplan_planid
				INNER JOIN ues.dbo.Bundles b ON b.bndl_id = bp.bndlplan_bndlid AND b.bndl_erid = @erid
				INNER JOIN ues.dbo.SelectionLists sl ON b.bndl_sellistid = sl.sellist_id AND sl.sellist_erid = @erid
				INNER JOIN ues.dbo.EligGroups eg ON sl.sellist_eliggrpid = eg.eliggrp_id AND eg.eliggrp_erid = @erid
				INNER JOIN ues.dbo.EmployeeElections eel ON eel.elect_sellistid = sl.sellist_id AND eel.elect_eliggrpid = eg.eliggrp_id
				INNER JOIN ues.dbo.DetectEvents d1 ON d1.detevt_eeid = eel.elect_eeid
				LEFT OUTER JOIN ues.dbo.ServiceAreaMasterMapping samm ON samm.samm_planid = p.plan_id
			WHERE p.plan_subtype = 'Custom Tier' AND bp.bndlplan_cobraeligibleplan = 1
				AND bp.bndlplan_effto_date > getdate() AND bp.bndlplan_erid = @erid
				AND eel.elect_eeid = @eeid
				AND d1.detevt_status = 1
				AND d1.detevt_type IN (35,23,14,55,5,6,12,21,22,11,13,15,120,56,19,62,121,63,122,76,20)
		END
	
    INSERT INTO @sellists2 (eliggrp_id, sellist_id, bndlplan_id)
    SELECT eliggrp_id, sellist_id, bndlplan_id
    FROM @sellists
    GROUP BY eliggrp_id, sellist_id, bndlplan_id
	UNION
    SELECT eliggrp_id, sellist_id, bndlplan_id
    FROM @sellistsEAP
    GROUP BY eliggrp_id, sellist_id, bndlplan_id
    
    /* Additional code for service area changes and active bundle plans */
```

This section continues data preparation:
- Adds EAP plans for life events after COBRA
- Combines and processes selection lists
- Identifies service area changes
- Filters out non-changes
- Retrieves active bundle plans

## Section 6: Event Processing - Standard Events (Lines 360-505)
```sql
  -- Three different queries need to be run to pull back the correct information.
  -- First, retrieve all events that are not:
  --    3 = Termination
  --    9 = Gain of Eligibility
  --   38 = Service Area Gain
  --   60 = Length of Service Gain of Eligibility
	IF (@elevatedDepedentWithAdminEvent = 1 OR @adminAfterCobraEvent = 1)
		BEGIN
			INSERT INTO @unionTable1
				    (evtId, evtEeId, evtErId, evtType, evtReasonId, evtSource, evtEligGrpId, evtDate,
					 evtDetDate, evtCause, eligGrpPlanYrEnd, procAction, evtActElectId, evtActAction,
					 evtActElectStatus, evtActEffDateTiming, evtActActionWindow, evtActLifeStatusChangeEffDateWaitPeriod, evtActBndlPlanId,
					 evtActRuleId, evtActSelListId, evtEffDate, causedEvtId, causedEvtType, causedEvtReasonId,
					 causedEvtSource, causedEvtEligGrpId, causedEvtDate, causedEvtDetDate, causedEvtAction,
					 causedEvtEffDateTiming, causedEvtWindow, causedEvtLSChangeEffDateWaitingPeriod, causedEvtRuleId,
					 causedEvtEffDate, evtFirstOfMonthRule, electTranDate)
			SELECT	de.detevt_id AS	evtId, de.detevt_eeid AS evtEeId, @erid AS evtErId, de.detevt_type AS evtType, de.detevt_reasonid AS evtReasonId, de.detevt_source AS evtSource,
					eg.eliggrp_id AS evtEligGrpId, de.detevt_evtdate AS evtDate, de.detevt_detdate AS evtDetDate, de.detevt_causeEvtId AS evtCause, eg.eliggrp_planyr_end AS eligGrpPlanYrEnd, 
					4 AS procAction, -- General Event Processor 
					0 AS evtActElectId, er.evtrule_action AS evtActAction, 0 AS evtActElectStatus, er.evtrule_effdatetiming AS evtActEffDateTiming, er.evtrule_window AS evtActActionWindow, 
					er.evtrule_eff_wait AS evtActLifeStatusChangeEffDateWaitPeriod, s.bndlplan_id AS evtActBndlPlanId, er.evtrule_id AS evtActRuleId, s.sellist_id AS evtActSelListId, 
					(dbo.GetEventEffectiveDate(de.detevt_evtdate, er.evtrule_effdatetiming, IsNull(er.evtrule_eff_wait,0), DEFAULT, er.evtrule_life_event_first_of_month_rule, @eeId)) AS evtEffDate,
					NULL AS causedEvtId, NULL AS causedEvtType, NULL AS causedEvtReasonId, NULL AS causedEvtSource, NULL AS causedEvtEligGrpId, NULL AS causedEvtDate, NULL AS causedEvtDetDate, 
					NULL AS causedEvtAction, NULL AS causedEvtEffDateTiming, NULL AS causedEvtWindow, NULL AS causedEvtLSChangeEffDateWaitingPeriod, NULL AS causedEvtRuleId, NULL AS causedEvtEffDate, 
					NULL AS evtFirstOfMonthRule, NULL as electTranDate
			FROM ues.dbo.DetectEvents de
				INNER JOIN @sellistsElevatedDependents AS s ON de.detevt_eeid = ee_id
				INNER JOIN ues.cobra.COBRAEEEligGroups ceeeg ON ceeeg.ceeeg_eeid = de.detevt_eeid AND s.eliggrp_id = ceeeg.ceeeg_eliggrpid 
				INNER JOIN ues.dbo.EligGroups eg ON ceeeg.ceeeg_eliggrpid = eg.eliggrp_id AND eg.eliggrp_erid = @erid			
				INNER JOIN ues.dbo.EventRulesLink erl ON (erl.erl_sellistid = s.sellist_id)
				INNER JOIN ues.dbo.EventRules er ON (er.evtrule_egid = erl.erl_egid AND de.detevt_type = er.evtrule_event AND de.detevt_reasonid = er.evtrule_reasonid)
			WHERE  de.detevt_eeid = @eeId
				AND eg.eliggrp_planyr_end >= de.detevt_evtdate
				AND de.detevt_status = 1
				AND de.detevt_type = 20
		END
    /* Additional conditions for other event types */
```

This section begins event processing for standard events (not terminations or service area changes):
- Uses different logic based on employee status
- Handles special cases for elevated dependents and admin events
- Ensures EAP plans are included for life events after COBRA

## Section 7: Event Processing - Service Area Events (Lines 506-706)
```sql
    -- Second, retrieve all events that are:
    --   38 = Service Area Gain and where the service area gain fits the plan rules for service area
	DECLARE @tblgetsams TABLE
			(ee_id INT, sellist_id INT, bndlplan_id INT, newzip CHAR(50), oldzip CHAR(50), inold INT, inNew INT)

	INSERT @tblgetsams (ee_id, sellist_id, bndlplan_id, newzip, oldzip, inold, inNew)
	SELECT sam.ee_id, sam.sellist_id, sam.bndlplan_id, MAX(sam.newzip), MAX(sam.oldzip), MAX(sam.inOld), MAX(sam.inNew)
	FROM (
		SELECT e.ee_id, eg.sellist_id, eg.bndlplan_id, e.pers_postal_code AS newzip, p.pers_postal_code AS oldzip,
		CASE WHEN samm.samm_id is null THEN 0 
			 WHEN EXISTS (SELECT 1 FROM ues.dbo.ServiceAreaMasterLocation samlp WHERE samlp.saml_samdrid = samdr.samdr_id AND p.pers_postal_code = samlp.saml_postalCode) THEN 1
			 WHEN EXISTS (SELECT 1 FROM ues.dbo.ServiceAreaMasterLocation samls WHERE samls.saml_samdrid = samdr.samdr_id AND p.pers_region = samls.saml_state) THEN 1 
			 ELSE 0 
		END AS inOld,
		CASE WHEN samm.samm_id is null THEN 1 
			 WHEN EXISTS (SELECT 1 FROM ues.dbo.ServiceAreaMasterLocation samlp WHERE samlp.saml_samdrid = samdr.samdr_id AND e.pers_postal_code = samlp.saml_postalCode) THEN 1
			 WHEN EXISTS (SELECT 1 FROM ues.dbo.ServiceAreaMasterLocation samls WHERE samls.saml_samdrid = samdr.samdr_id AND e.pers_region = samls.saml_state) THEN 1 
			 ELSE 0 
		END AS inNew
		FROM  @sellists AS eg
		INNER JOIN ues.dbo.vEmployee e ON (e.ee_id = @eeId)
		INNER JOIN ues.dbo.Person p ON (p.rec_orig_id = e.ee_personid AND p.rec_end_date = e.pers_rec_start_date)
		INNER JOIN ues.dbo.EligGroups egrp ON egrp.eliggrp_id = eg.eliggrp_id
		LEFT OUTER JOIN ues.dbo.ServiceAreaMasterMapping samm ON samm.samm_planid = eg.plan_id
		LEFT OUTER JOIN ues.dbo.ServiceAreaMasterDateRange samdr ON samdr.samdr_samid = samm.samm_samid
		WHERE  e.ee_id = @eeid
			AND e.ee_erid = @erid
			AND egrp.eliggrp_planyr_beg >= samdr.samdr_effectiveStartDate
			AND egrp.eliggrp_planyr_beg <= samdr.samdr_effectiveEndDate
			AND EXISTS (SELECT 1 FROM @sagSellists sl WHERE sl.sag_sellistid = eg.sellist_id)
		) sam
	GROUP BY sam.ee_id, sam.sellist_id, sam.bndlplan_id

    /* Additional code for service area gain events */
    /* Code for service area gain to default plans */
    /* Code for service area loss events */
```

This section processes service area change events:
- Creates temporary tables to track service area changes
- Processes service area gain events (type 38)
- Processes service area gain events specifically for default plans
- Processes service area loss events (type 37)

## Section 8: Event Processing - Termination Events (Lines 707-822)
```sql
  -- Third, retrieve all termination actions, using the default rules if the specific reason code is not set up

    INSERT INTO @unionTable1
			(evtId, evtEeId, evtErId, evtType, evtReasonId, evtSource, evtEligGrpId, evtDate,
			evtDetDate, evtCause, eligGrpPlanYrEnd, procAction, evtActElectId, evtActAction,
			evtActElectStatus, evtActEffDateTiming, evtActActionWindow, evtActLifeStatusChangeEffDateWaitPeriod, evtActBndlPlanId,
			evtActRuleId, evtActSelListId, evtEffDate, causedEvtId, causedEvtType, causedEvtReasonId,
			causedEvtSource, causedEvtEligGrpId, causedEvtDate, causedEvtDetDate, causedEvtAction,
			causedEvtEffDateTiming, causedEvtWindow, causedEvtLSChangeEffDateWaitingPeriod, causedEvtRuleId,
			causedEvtEffDate, evtFirstOfMonthRule, electTranDate)
    SELECT	de.detevt_id AS evtId, de.detevt_eeid AS evtEeId, ee.ee_erid AS evtErId, de.detevt_type AS evtType, de.detevt_reasonid AS evtReasonId, de.detevt_source AS evtSource, 
			de.detevt_eliggrpid AS evtEligGrpId, de.detevt_evtdate AS evtDate, de.detevt_detdate AS evtDetDate, de.detevt_causeEvtId AS evtCause, eg.eliggrp_planyr_end AS eligGrpPlanYrEnd, 
			4 AS procAction, -- General Event Processor 
			el.elect_id AS evtActElectId, (CASE WHEN er.evtrule_action IS NULL THEN erDefault.evtrule_action ELSE er.evtrule_action END) AS evtActAction, el.elect_status AS evtActElectStatus, 
			(CASE WHEN er.evtrule_effdatetiming IS NULL THEN erDefault.evtrule_effdatetiming ELSE er.evtrule_effdatetiming END) AS evtActEffDateTiming, 
			(CASE WHEN er.evtrule_window IS NULL THEN erDefault.evtrule_window ELSE er.evtrule_window END) AS evtActActionWindow, 
			(CASE WHEN er.evtrule_eff_wait IS NULL THEN erDefault.evtrule_eff_wait ELSE er.evtrule_eff_wait END) AS evtActLifeStatusChangeEffDateWaitPeriod, 
			el.elect_bndlplanid AS evtActBndlPlanId, (CASE WHEN er.evtrule_id IS NULL THEN erDefault.evtrule_id ELSE er.evtrule_id END) AS evtActRuleId, el.elect_sellistid AS evtActSelListId,
			ues.dbo.fnMaxDate((CASE WHEN er.evtrule_id IS NULL
									THEN ues.dbo.GetTermEffectiveDate(de.detevt_evtdate, erDefault.evtrule_effdatetiming, erDefault.evtrule_eff_wait)
									ELSE ues.dbo.GetTermEffectiveDate(de.detevt_evtdate, er.evtrule_effdatetiming, IsNull(er.evtrule_eff_wait,0))
								END),
								DATEADD(day, -1, eg.eliggrp_planyr_beg)) AS evtEffDate, 
			NULL AS causedEvtId, NULL AS causedEvtType, NULL AS causedEvtReasonId, NULL AS causedEvtSource, NULL AS causedEvtEligGrpId, NULL AS causedEvtDate, NULL AS causedEvtDetDate, NULL AS causedEvtAction, 
			NULL AS causedEvtEffDateTiming, NULL AS causedEvtWindow, NULL AS causedEvtLSChangeEffDateWaitingPeriod, NULL AS causedEvtRuleId, NULL AS causedEvtEffDate, er.evtrule_first_of_month_rule AS evtFirstOfMonthRule, 
			NULL as electTranDate
    /* Query conditions */

  -- For wellness, retrieve all termination actions, using the default rules if the specific reason code is not set up
  /* Additional wellness termination event processing */
  /* Other wellness event processing */
```

This section processes termination events:
- Handles standard termination events (type 3)
- Processes wellness program termination events
- Processes other wellness program events

## Section 9: Handling Special Plan Types (Lines 823-874)
```sql
  -- Yet another select: this time to force HSA (plan type 8) and employer contributions (planType 11) and FSA-Limited (planType 4) to be included even when the employee doesn't have an election
  	IF NOT (@lifeAfterCobraEventExists = 1 AND @employeeTerminatedStatus = 1)
		BEGIN
		    INSERT INTO @unionTable1
				(evtId, evtEeId, evtErId, evtType, evtReasonId, evtSource, evtEligGrpId, evtDate,
				evtDetDate, evtCause, eligGrpPlanYrEnd, procAction, evtActElectId, evtActAction,
				evtActElectStatus, evtActEffDateTiming, evtActActionWindow, evtActLifeStatusChangeEffDateWaitPeriod, evtActBndlPlanId,
				evtActRuleId, evtActSelListId, evtEffDate, causedEvtId, causedEvtType, causedEvtReasonId,
				causedEvtSource, causedEvtEligGrpId, causedEvtDate, causedEvtDetDate, causedEvtAction,
				causedEvtEffDateTiming, causedEvtWindow, causedEvtLSChangeEffDateWaitingPeriod, causedEvtRuleId,
				causedEvtEffDate, evtFirstOfMonthRule, electTranDate)
		    SELECT	de.detevt_id AS evtId, de.detevt_eeid AS evtEeId, ee.ee_erid AS evtErId, de.detevt_type AS evtType, de.detevt_reasonid AS evtReasonId, de.detevt_source AS evtSource, 
					de.detevt_eliggrpid AS evtEligGrpId, de.detevt_evtdate AS evtDate, de.detevt_detdate AS evtDetDate, de.detevt_causeEvtId AS evtCause, eg.eliggrp_planyr_end AS eligGrpPlanYrEnd, 
					4 AS procAction, 0 AS evtActElectId, er.evtrule_action AS evtActAction, 0 AS evtActElectStatus, er.evtrule_effdatetiming AS evtActEffDateTiming, er.evtrule_window AS evtActActionWindow, 
					er.evtrule_eff_wait AS evtActLifeStatusChangeEffDateWaitPeriod, s.bndlplan_id AS evtActBndlPlanId, er.evtrule_id AS evtActRuleId, s.sellist_id AS evtActSelListId, 
					/* Additional fields and conditions */
		    FROM ues.dbo.DetectEvents de
				/* Joins and conditions for special plan types */
	END
```

This section ensures special plan types are included in the results:
- HSA plans (plan_type = 8)
- Employer contribution plans (plan_type = 11)
- FSA-Limited plans (plan_type = 4 with subtype 'FSA - Limited')
- Only processes if not a life event after COBRA for terminated employee

## Section 10: Leave of Absence (LOA) Processing (Lines 875-994)
```sql
    -- BEGIN LOA Actions Processing
    IF @loaSetupExists = 1
    BEGIN
        DELETE FROM @unionTable1 WHERE evtType IN (33,1103)

        INSERT INTO @unionTable1
			(evtId, evtEeId, evtErId, evtType, evtReasonId, evtSource, evtEligGrpId, evtDate,
			evtDetDate, evtCause, eligGrpPlanYrEnd, procAction, evtActElectId, evtActAction,
			evtActElectStatus, evtActEffDateTiming, evtActActionWindow, evtActLifeStatusChangeEffDateWaitPeriod, evtActBndlPlanId,
			evtActRuleId, evtActSelListId, evtEffDate, causedEvtId, causedEvtType, causedEvtReasonId,
			causedEvtSource, causedEvtEligGrpId, causedEvtDate, causedEvtDetDate, causedEvtAction,
			causedEvtEffDateTiming, causedEvtWindow, causedEvtLSChangeEffDateWaitingPeriod, causedEvtRuleId,
			causedEvtEffDate, evtFirstOfMonthRule, electTranDate)
       SELECT	de.detevt_id AS evtId, de.detevt_eeid AS evtEeId, @ee_erid AS evtErId, de.detevt_type AS evtType, de.detevt_reasonid AS evtReasonId, de.detevt_source AS evtSource,
				de.detevt_eliggrpid AS evtEligGrpId, 
				CASE
					WHEN de.detevt_type = 1103
						THEN de.detevt_evtdate
					WHEN loalds.loalds_id = 1 -- original LOA leave date
						THEN loaold.loaold_original_leave_date
					WHEN loalds.loalds_id = 3 -- Return to Active Date
						THEN @eed_active_status_date
					WHEN loalds.loalds_id = 4 -- Enrollment Date
						THEN GETDATE()
					ELSE de.detevt_evtdate -- current leave date
				END AS evtDate,
				/* Additional fields and complex logic */
        /* UNION with additional LOA processing for wellness programs */
    END;
    -- END LOA Actions Processing 
```

This section handles Leave of Absence events:
- Only processes if LOA setup exists
- Removes any existing LOA events from the results
- Inserts events specific to LOA processing
- Handles both standard selections and wellness program selections

## Section 11: Result Selection and Final Processing (Lines 995-1064)
```sql
    SELECT DISTINCT
        evtId, evtEeId, evtErId, evtType, evtReasonId, evtSource, evtEligGrpId, evtDate, evtDetDate, evtCause,
        eligGrpPlanYrEnd, procAction, evtActElectId, evtActAction, evtActElectStatus, evtActEffDateTiming, evtActActionWindow,
        evtActLifeStatusChangeEffDateWaitPeriod, evtActBndlPlanId, evtActRuleId, evtActSelListId, evtEffDate, causedEvtId, causedEvtType,
        causedEvtReasonId, causedEvtSource, causedEvtEligGrpId, causedEvtDate, causedEvtDetDate, causedEvtAction, causedEvtEffDateTiming,
        causedEvtWindow, causedEvtLSChangeEffDateWaitingPeriod, causedEvtRuleId, causedEvtEffDate, evtFirstOfMonthRule,electTranDate
    FROM @unionTable1
	ORDER BY evtId, evtActSelListId, evtActBndlPlanId, evtActElectStatus, evtActRuleId,electTranDate desc

	/* hack: enddate superfluous waives so "- No waives" event actions work*/
	IF @sagain_evtdt IS NOT NULL
	BEGIN
		SELECT @sellistId = 0,@electId = 0
		IF EXISTS (SELECT 1 FROM @activeBndlplans)
		BEGIN
			WHILE EXISTS(SELECT 1 FROM @activeBndlplans WHERE sellistId > @sellistId)
			BEGIN
				SELECT TOP (1)  @sellistId = sellistId FROM @activeBndlplans WHERE sellistId > @sellistId ORDER BY sellistId
				SET @electId = 0
				SELECT @electId = elect_id FROM ues.dbo.employeeelections WHERE elect_status = 5 AND elect_effto_date >= elect_curr_eff_date AND elect_eeid = @eeid AND elect_sellistId = @sellistId
				-- print 'electId ='+ convert(varchar,@electId) +',sellistId ='+ convert(varchar,@sellistId)
				IF ISNULL(@electId,0) > 0
				BEGIN
					UPDATE ues.dbo.employeeelections SET elect_effto_date = dateadd(day,-1,elect_curr_eff_date), elect_tran_date = convert(VARCHAR(19),getdate(),121) WHERE elect_id = @electId AND elect_status = 5 AND elect_eeid = @eeid
					EXEC ues.dbo.sp_insert_elhist @electId ,''
				END
			END
		END
	END
END
```

This section finalizes the result set:
- Returns distinct events/actions with a specific sort order
- Performs special handling for "superfluous waives" if service area gain events exist
- Updates end dates for waives to ensure proper functionality

## Section 12: Footer and Permissions (Lines 1065-1066)
```sql
/* Sample Execution
    EXEC ues.dbo.GetEventsWithActionsByEeId 92
    EXEC GetEventsWithActionsByEeId 9209756
    EXEC #GetEventsWithActionsByEeId 9209756
*/

-- =====================================================================================================================
-- Set permissions
-- =====================================================================================================================
GRANT EXECUTE ON [dbo].[GetEventsWithActionsByEeId] TO exec_proc;
GO
```

This section includes sample execution examples and sets the appropriate permissions.

## Summary

The stored procedure `GetEventsWithActionsByEeId` is a complex procedure that:

1. Takes an employee ID as input
2. Determines the employee's status (terminated, COBRA, LOA, etc.)
3. Gathers relevant selection lists and plan information
4. Processes multiple event types:
   - Standard life events
   - Service area changes (gains and losses)
   - Terminations
   - Leave of absence events
5. Handles special plan types (HSA, FSA-Limited, employer contributions)
6. Ensures proper handling of wellness programs
7. Returns a comprehensive list of events with their associated actions

The procedure is heavily optimized with performance considerations throughout, and has been refined over many years as evidenced by the extensive update history in the comments.
