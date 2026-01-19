# Upgrade Instructions
With the exception of the cases reported here, you should remove the dashboard completely and re-deploy the new version.

## Upgrade to v4.0.2

### Upgrade to v4.0.2 from v4.0.1
This version has a new `config_compliance` view, which can be updated on the Amazon Athena Console.

1. Open the Amazon Athena Console on the Log Archive account, in the region where you deployed the dashboard.
1. Select the `cid_crcd_database` database.
1. Execute the following SQL on a new Query tab (the dashboard will show correct data after the next scheduled dataset refresh).

```sql
CREATE OR REPLACE VIEW "config_compliance" AS 
WITH
  conformance_packs AS (
   SELECT
     ARBITRARY("configurationItem"."resourceId") "ConformancePackId"
   , ARBITRARY("configurationItem"."resourceName") "ConformancePackName"
   , "json_extract_scalar"(rule, '$.configRuleName') "ConformancePackRuleName"
   FROM
     ((cid_crcd_config
   CROSS JOIN UNNEST("configurationitems") t (configurationItem))
   CROSS JOIN UNNEST(CAST("json_extract"("configurationItem"."configuration", '$.configRuleList') AS array(json))) u (rule))
   WHERE (("configurationItem"."resourcetype" = 'AWS::Config::ConformancePackCompliance') AND (((CAST(date_parse("dt", '%Y-%m-%d') AS "Date") = date_add('day', -1, current_date)) OR (CAST(date_parse("dt", '%Y-%m-%d') AS "Date") = last_day_of_month(CAST(date_parse("dt", '%Y-%m-%d') AS "Date")))) AND (CAST(date_parse("dt", '%Y-%m-%d') AS "Date") >= last_day_of_month(date_add('month', -5, current_date)))))
   GROUP BY "json_extract_scalar"(rule, '$.configRuleName')
) 
, resource_compliance AS (
   SELECT
     "accountId" "AccountId"
   , "region" "Region"
   , CAST(date_parse("dt", '%Y-%m-%d') AS "Date") "ComplianceSampleDate"
   , MAX(CAST(parse_datetime("configurationItem"."configurationitemcapturetime", 'yyyy-MM-dd''T''HH:mm:ss.SSSZ') AS timestamp)) "ConfigItemCaptureTime"
   , MAX_BY("configurationItem"."resourceId", "configurationItem"."configurationitemcapturetime") "ConfigEventResourceId"
   , MAX_BY("json_extract_scalar"("configurationItem"."configuration", '$.targetResourceType'), "configurationItem"."configurationitemcapturetime") "ResourceType"
   , MAX_BY("json_extract_scalar"(rule, '$.complianceType'), "configurationItem"."configurationitemcapturetime") "ComplianceType"
   , "json_extract_scalar"(rule, '$.configRuleId') "RuleId"
   , ARBITRARY("json_extract_scalar"(rule, '$.configRuleName')) "RuleName"
   , "json_extract_scalar"("configurationItem"."configuration", '$.targetResourceId') "ResourceId"
   FROM
     ((cid_crcd_config
   CROSS JOIN UNNEST("configurationitems") t (configurationItem))
   CROSS JOIN UNNEST(CAST("json_extract"("configurationItem"."configuration", '$.configRuleList') AS array(json))) u (rule))
   WHERE (("configurationItem"."resourcetype" = 'AWS::Config::ResourceCompliance') AND (((CAST(date_parse("dt", '%Y-%m-%d') AS "Date") = date_add('day', -1, current_date)) OR (CAST(date_parse("dt", '%Y-%m-%d') AS "Date") = last_day_of_month(CAST(date_parse("dt", '%Y-%m-%d') AS "Date")))) AND (CAST(date_parse("dt", '%Y-%m-%d') AS "Date") >= last_day_of_month(date_add('month', -5, current_date)))) AND (datasource = 'ConfigSnapshot'))
   GROUP BY "AccountId", "Region", "dt", "json_extract_scalar"(rule, '$.configRuleId'), "json_extract_scalar"("configurationItem"."configuration", '$.targetResourceId')
) 
SELECT
  rc.AccountId
, rc.Region
, rc.ComplianceSampleDate
, rc.ConfigItemCaptureTime
, rc.ConfigEventResourceId
, rc.ResourceType
, rc.ComplianceType
, rc.RuleId
, rc.RuleName
, (CASE WHEN REGEXP_LIKE("RuleName", 'AWSControlTower') THEN "RuleName" WHEN REGEXP_LIKE("RuleName", 'securityhub') THEN REGEXP_REPLACE("RuleName", '-[^-]*$', '') WHEN (cp.ConformancePackId IS NULL) THEN "RuleName" ELSE REGEXP_REPLACE("RuleName", '-[^-]*$', '') END) "NormalizedRuleName"
, rc.ResourceId
, cp.ConformancePackId
, cp.ConformancePackName
, (CASE WHEN (NOT REGEXP_LIKE("ConformancePackName", 'OrgConformsPack')) THEN "ConformancePackName" ELSE REGEXP_REPLACE("ConformancePackName", '-[^-]*$', '') END) "NormalizedConformancePackName"
, concat(concat(concat(concat(rc.AccountId, '-'), rc.Region), '-'), rc.ResourceId) "PKResourceId"
, concat(concat(concat(concat(rc.AccountId, '-'), rc.Region), '-'), rc.RuleId) "PKRuleId"
FROM
  (resource_compliance rc
LEFT JOIN conformance_packs cp ON (cp.ConformancePackRuleName = rc.RuleName))
```

## Upgrade to v4.0.2 from earlier versions
You have to destroy the resources of the current versions and redeploy.

## Upgrade to v4.0.1
You have to destroy the resources of the current versions and redeploy.

## Upgrade to v4.0.0
You have to destroy the resources of the current versions and redeploy.

## Upgrade to v3.0.1
There are no functional changes over v3.0.0. The new version allows CRCD to be easier to install together with other CID dashboards, if you are already running v3.0.0 there is no reason to upgrade.

## Upgrade to v3.0.0

### Upgrade to v3.0.0 from v2.1.1 or v2.1.0
You only need to redeploy the frontend resources with the `cid-cmd` tool. You can keep the data pipeline resources that were installed by CloudFormation, as is.

#### Step 1: Enable AWS Config history files on the Lambda Partitioner function
1. Open the Lambda Console on the AWS account and region where you deployed the dashboard.
1. Select the Lambda Partitioner function, it's called `crcd-config-file-partitioner`.
1. Ensure the [environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html) `PARTITION_CONFIG_SNAPSHOT_RECORDS` and `PARTITION_CONFIG_HISTORY_RECORDS` are both set to `1`.

#### Step 2: Uninstall the dashboard frontend with the cid-cmd tool
1. On the same AWS account and region, open Amazon CloudShell
1. Execute the following command to delete the dashboard:

```
cid-cmd delete --resources cid-crcd.yaml
```

* `cid-crcd.yaml` is the template file for the dashboard resources. Upload it to CloudShell if needed.

3. When prompted:
   - Select the `[cid-crcd] AWS Config Resource Compliance Dashboard (CRCD)` dashboard.
   - For each QuickSight dataset, choose `yes` to delete the dataset.
   - Accept the default values for the S3 Path for the Athena table.
   - Accept the default values for the four tags.
   - For each Athena view, choose `yes` to delete the dataset.

#### Step 3: Deploy the dashboard frontend again
Follow the installation steps to deploy the dashboard resources using the `cid-cmd` tool. This is either **Step 2** in case of Log Archive account deployment, or **Step 3** in case of Dashboard account deployment.

#### Step 4: Optionally change the retention period of the Lambda Partitioner CloudWatch logs
AWS Config Dashboard v2.2.1 and v2.1.0 did not configure a retention period for the CloudWatch logs of the Lambda Partitioner function. From version 3.0.0, logs are kept for 14 days. We recommend that you configure a retention period for your logs. Follow these steps:

1. Open the CloudWatch Console on the AWS account and region where you deployed the dashboard.
1. Click on `Logs/Log groups`.
1. Find the log group called `/aws/lambda/crcd-config-file-partitioner`.
1. Edit the log settings, change retention to 14 days, or the value that suits your needs.

### Upgrade to v3.0.0 from older versions
You have to destroy the resources of the current versions and redeploy. After you removed the old versions, and before deploying v3.0.0, make sure to delete the CloudWatch log group of the Lambda Partitioner:
1. Log onto the AWS Console on the account and region where you deploy the dashboard, open the CloudWatch console.
1. Click on `Logs/Log groups`.
1. Find the log group called `/aws/lambda/crcd-config-file-partitioner` and select it.
1. Click on the `Actions` button and select `Delete log group(s)`.

The AWS Config Dashboard v3.0.0 creates the CloudWatch log group as a CloudFormation resource with a retention period, the installation will fail if the log group already exists.