---
title: "Azure Backups for Virtual Machines and SQL"
date: 2021-04-29T15:00:00+12:00
draft: false
tags: ["azure","azure backup","powershell","virtual machine","sql"]
---

Backups are a necessity but it still amazes me the amount of companies I have worked at where you find backups are just not done, or just as bad, are done but are never checked!

There are multiple ways to do backups depending where your data lives.  Veeam, BackupExec, etc, but if you are running your infrastructure in Azure and you are using Virtual Machines then [Azure Backup](https://docs.microsoft.com/en-us/azure/backup/backup-overview) is an amazing tool for this.

You can setup Azure Backup via the Portal, and this is great if you are doing this once or twice, but what if you are doing this at scale for multiple accounts?   **_Powershell is the answer_**....  I created a script one day when I realized I was going to be setting up backups more than once, and I have used this script now at least 80 times, saving me so much time.  Its pretty much set a few variables up front, execute the script, get a coffee, come back 20 minutes later and you have a full backup solution.   The extra benefit of this is that a lot of the clients I have worked at have SQL Server running inside a Windows VM still, alongside the usual Azure SQL.    Azure Backup if you tell it too, will backup the SQL Server in a VM and do per database level backups which you can easily restore from which is super handy.

You can find the script in [my github repo here](https://github.com/andyr8939/Powershell/blob/master/azure/Setup-AzureVMBackups.ps1) but I'll give you the overview of it below.

## Requirements

You are going to need a few things first.
1. An Azure Subscription
2. A Resource Group
3. Some Virtual Machines running in the resource group, windows or linux, it doesn't matter.
4. Optional but if you have SQL Server on a VM, this script also covers that.

## Initial Variable

These are set at the top of the script and include subscription details, resource group name, the backup time you want to execute at etc.  I put everything at the top like this so you can just edit one section and that it, plus you can also then have these variables replaced in any other automation (Octopus Deploy) or CI/CD process (Azure DevOps Pipeline) you use to make this even more streamlined.

```powershell
########################################################################################
## Connection Details
$mySubscription = "123456789-1234-1234-1234-123456789"
$myLocation = "australiaeast"
$myClient = "myclient"
$myEnvironment = "test"
$myRSG = "$myEnvironment-$myLocation-$myClient"
$myBackupVaultName = "$myEnvironment-$myLocation-$myClient-backup-vault"
$myBackupTime = Get-Date -Date "2021-02-1 22:00:00Z" # This gets converted to UTC for local 1am
########################################################################################
# Set your Global Details Here
$myBackupPolicyName = "MyBackupPolicy"
$mySQLBackupPolicyName = "MySQLBackupPolicy"

# VM Backup Cycle
$DailyBackupCount = 7
$WeeklyBackupCount = 4
$MonthlyBackupCount = 12
$YearlyBackupCount = 2
$MonthlyDayOfWeek = "Sunday"
$MonthlyWeek = "Last"
$YearlyMonth = "January"

# SQL in VM Specific Cycle
$sqlFullBackupFrequency = "Weekly"
$sqlFullBackupFrequencyDay = "Sunday"
$sqlFullBackupWeekOfMonth = "First"
$sqlWeeklyCount = "4"
$sqlMonthlyCount = "12"
$sqlYearlyCount = "2"
$sqlYearlyMonth = "January"

$sqlDifferentialEnabled = "True"
$sqlLogBackupEnabled = "True"
$sqlLogRetentionCycle = "Days"
$sqlLogRetentionCount = "7"
$sqlLogBackupFrequencyMins = "120" # Take Log Backup every 2 hours

########################################################################################
```

There looks a lot there but once you get going its pretty easy to replicate this.  Pretty much all I ever need to change between clients is the connection details at the top.  This setup then gives you the typical [Grandfather-Father-Son](https://en.wikipedia.org/wiki/Backup_rotation_scheme#Grandfather-father-son) style backup process.

Once you have populated those details, that really is it.  You can then run the script and it will set everything up for you, but for an explanation of whats happening lets break it down.

### Script Breakdown

The first part of executing the script connects to your Azure Account, gets the VMs in your Resource Group and then sets up the Recovery Services Vault which is where all your backups will be held.

```powershell
# Set Azure Context
Set-AzContext -SubscriptionId $mySubscription

# Get a list of all VMs in the Resource Group
$VMs = Get-AzVm -ResourceGroupName $myRSG

# Register Recovery Services Providor
Register-AzResourceProvider -ProviderNamespace "Microsoft.RecoveryServices"

# Create Backup RecoveryServices Vault - This is different to the replication Vault as needs to be in the same region as the VMs
New-AzRecoveryServicesVault -Name $myBackupVaultName -ResourceGroupName $myRSG -Location $myLocation
$backup_vault = Get-AzRecoveryServicesVault -Name $myBackupVaultName -ResourceGroupName $myRSG

# Set vault redundancy
Set-AzRecoveryServicesBackupProperty -Vault $backup_vault -BackupStorageRedundancy GeoRedundant

# Set vault context
Get-AzRecoveryServicesVault -Name $myBackupVaultName -ResourceGroupName $myRSG | Set-AzRecoveryServicesVaultContext

# Get vault ID
$targetVault = Get-AzRecoveryServicesVault -ResourceGroupName $myRSG -Name $myBackupVaultName
```

After this we will setup the VM Backup Policy with the schedules we specified in the earlier variables.

```powershell
# Set Values for the Protection Policy
$myBackupPolicyName = $myBackupPolicyName
$schPol = Get-AzRecoveryServicesBackupSchedulePolicyObject -WorkloadType "AzureVM"
$UtcTime = $myBackupTime.ToUniversalTime()
$schpol.ScheduleRunTimes[0] = $UtcTime

# Create the Protection Policy
$retPol = Get-AzRecoveryServicesBackupRetentionPolicyObject -WorkloadType "AzureVM"
New-AzRecoveryServicesBackupProtectionPolicy -Name $myBackupPolicyName -WorkloadType "AzureVM" -RetentionPolicy $retPol -SchedulePolicy $schPol

# Modify protection policy now its in place
$retPol.DailySchedule.DurationCountInDays = $DailyBackupCount
$retPol.WeeklySchedule.DurationCountInWeeks = $WeeklyBackupCount
$retPol.MonthlySchedule.DurationCountInMonths = $MonthlyBackupCount
$retPol.MonthlySchedule.RetentionScheduleWeekly.DaysOfTheWeek = $MonthlyDayOfWeek
$retPol.MonthlySchedule.RetentionScheduleWeekly.WeeksOfTheMonth = $MonthlyWeek
$retPol.YearlySchedule.DurationCountInYears = $YearlyBackupCount
$retPol.YearlySchedule.MonthsOfYear = $YearlyMonth
$pol = Get-AzRecoveryServicesBackupProtectionPolicy -Name $myBackupPolicyName
Set-AzRecoveryServicesBackupProtectionPolicy -RetentionPolicy $RetPol -Policy $pol -SchedulePolicy $schPol
```

And now we will iterate through each of the VMs you have running in the resource group and enable them for backup with the policy we just created.  This part can vary depending on how many VMs you have, but I've seen it take 2-3mins per VM as a baseline.

```powershell
foreach ($vm in $VMs) {
    $pol = Get-AzRecoveryServicesBackupProtectionPolicy -Name $myBackupPolicyName
    Enable-AzRecoveryServicesBackupProtection -Policy $pol -Name $vm.Name -ResourceGroupName $myRSG
}
```

If you are only backing up vanilla VMs, you are now done.  Simple as that, your VMs will now backup and show in the Recovery Service Vault.

### Backing up Azure SQL in VM

The script will also continue if you have SQL Server running in a VM.  Even with the popularity of Azure SQL and SQL Managed Instances, I still come across a lot of companies using SQL in a VM due to legacy app requirements, so its important to capture this.

First it will create the SQL backup policy, enabling Log backups if you set it earlier in the variables, and likewise doing daily differentials.

```powershell
$sqlschPol = Get-AzRecoveryServicesBackupSchedulePolicyObject -WorkloadType "MSSQL"
$sqlretPol = Get-AzRecoveryServicesBackupRetentionPolicyObject -WorkloadType "MSSQL"
$SQLmyBackupTime = ($myBackupTime).AddHours(3) # Set to 3hr after the backup job time
$sqlUtcTime = $SQLmyBackupTime.ToUniversalTime()
$sqlschpol.FullBackupSchedulePolicy.ScheduleRunTimes[0] = $sqlUtcTime
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunTimes[0] = $sqlUtcTime
$sqlschpol.IsCompression = $NULL
$sqlschpol.FullBackupSchedulePolicy.ScheduleRunFrequency = $sqlFullBackupFrequency
$sqlschpol.FullBackupSchedulePolicy.ScheduleRunDays = $sqlFullBackupFrequencyDay
$sqlschpol.FullBackupSchedulePolicy.ScheduleRunTimes[0] = $sqlUtcTime
$sqlretPol.FullBackupRetentionPolicy.IsDailyScheduleEnabled = $NULL # This has to be null, not false like the other scheduleenabled ones! Goodbye 2hrs figuring that out!
$sqlretPol.FullBackupRetentionPolicy.DailySchedule = $NULL
$sqlretPol.FullBackupRetentionPolicy.WeeklySchedule.DurationCountInWeeks = $sqlWeeklyCount
$sqlretPol.FullBackupRetentionPolicy.WeeklySchedule.DaysOfTheWeek = $sqlFullBackupFrequencyDay
$sqlretPol.FullBackupRetentionPolicy.MonthlySchedule.DurationCountInMonths = $sqlMonthlyCount
$sqlretPol.FullBackupRetentionPolicy.MonthlySchedule.RetentionScheduleFormatType = $sqlFullBackupFrequency
$sqlretPol.FullBackupRetentionPolicy.MonthlySchedule.RetentionScheduleDaily = $NULL
$sqlretPol.FullBackupRetentionPolicy.MonthlySchedule.RetentionScheduleWeekly.DaysOfTheWeek = $sqlFullBackupFrequencyDay
$sqlretPol.FullBackupRetentionPolicy.MonthlySchedule.RetentionScheduleWeekly.WeeksOfTheMonth = $sqlFullBackupWeekOfMonth
$sqlretPol.FullBackupRetentionPolicy.YearlySchedule.DurationCountInYears = $sqlYearlyCount
$sqlretPol.FullBackupRetentionPolicy.YearlySchedule.RetentionScheduleFormatType = $sqlFullBackupFrequency
$sqlretPol.FullBackupRetentionPolicy.YearlySchedule.RetentionScheduleWeekly.DaysOfTheWeek = $sqlFullBackupFrequencyDay
$sqlretPol.FullBackupRetentionPolicy.YearlySchedule.RetentionScheduleWeekly.WeeksOfTheMonth = $sqlFullBackupWeekOfMonth
$sqlretPol.FullBackupRetentionPolicy.YearlySchedule.MonthsOfYear = $sqlYearlyMonth
$sqlschPol.IsDifferentialBackupEnabled = $sqlDifferentialEnabled
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunFrequency = $sqlFullBackupFrequency
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunDays = "Monday"
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunDays += "Tuesday"
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunDays += "Wednesday"
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunDays += "Thursday"
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunDays += "Friday"
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunDays += "Saturday"
$sqlschpol.DifferentialBackupSchedulePolicy.ScheduleRunTimes[0] = $sqlUtcTime
$sqlretPol.DifferentialBackupRetentionPolicy.RetentionDurationType = "Days"
$sqlretPol.DifferentialBackupRetentionPolicy.RetentionCount = "7"
$sqlschpol.IsLogBackupEnabled = $sqlLogBackupEnabled
$sqlschpol.LogBackupSchedulePolicy.ScheduleFrequencyInMins = $sqlLogBackupFrequencyMins
$sqlretPol.LogBackupRetentionPolicy.RetentionCount = $sqlLogRetentionCount
$sqlretPol.LogBackupRetentionPolicy.RetentionDurationType = $sqlLogRetentionCycle

# Create the SQL Backup Job
$NewSQLPolicy = New-AzRecoveryServicesBackupProtectionPolicy -Name $mySQLBackupPolicyName -WorkloadType "MSSQL" -RetentionPolicy $sqlretPol -SchedulePolicy $sqlschPol -Verbose
```

Next we get all the VMs that are running SQL.  Depending on our environment you could look for VM image type or extensions installed, but for me it was enough to simply find all VMs with _sql_ in the name.   Then once we have these we create a null config for the SQL IaaS Backup Extension to avoid conflicts and then disable it on each VM by applying the null config.  A null config is used to avoid changing other settings on the IaaS extension.

```powershell
# Get SQL VMs
$sqlvms = Get-AzVM -ResourceGroupName $myRSG -Name *sql*

# Register VMs with SQL on against the backup container
# And Disable SQL IaaS backup extension otherwise it causes conflicts
# Create a null auto backup config to disable IaaS backup
$autobackupconfig = New-AzVMSqlServerAutoBackupConfig -ResourceGroupName $myRSG

# This takes about 5 mins per VM
foreach ($vm in $sqlvms) {
    Register-AzRecoveryServicesBackupContainer -ResourceId $vm.ID -BackupManagementType AzureWorkload -WorkloadType MSSQL -VaultId $targetVault.ID -Force
    Set-AzVMSqlServerExtension -AutoBackupSettings $autobackupconfig -VMName $vm.Name -ResourceGroupName $myRSG -ErrorAction SilentlyContinue -WarningAction SilentlyContinue
}
```

Finally you will enable backup on all SQL VMs, take a backup of each DB, and most importantly enable auto protection for future DBs.  If you didnt turn this on and added another database later on, it wouldn't automatically be backed up.  But enabling auto protection the backup agent will rescan the server every 8 hours and add any missing databases in.

```powershell
# Get all protectable SQL DBs found in the vault that can be backed up
$SQLDB = Get-AzRecoveryServicesBackupProtectableItem -WorkloadType MSSQL -VaultId $targetVault.ID

# Enable backups on all DBs not already set to backup
foreach ($db in $SQLDB) {
    Enable-AzRecoveryServicesBackupProtection -ProtectableItem $db -Policy $NewSQLPolicy
}

# Now enable auto protection for all future DBs
# Get the SQL Instance Items
$SQLInstance = Get-AzRecoveryServicesBackupProtectableItem -workloadType MSSQL -ItemType SQLInstance -VaultId $targetVault.ID

# Set Auto Protect for each Instance
# This is then auto executed as a background task every 8hrs
foreach ($instance in $SQLInstance) {
    Enable-AzRecoveryServicesBackupAutoProtection -InputItem $instance -BackupManagementType AzureWorkload -WorkloadType MSSQL -Policy $NewSQLPolicy -VaultId $targetvault.ID
}
```

And thats it.  All your VMs in your Resource Group, along with SQL Databases living on a SQL in VM will be backed up to a new Recovery Services Vault.  I have been using this script for a couple of years now for well over 80 deployments and its not let me down yet.  Its probably longer than it needs to be, but I find it easily readable and easy to modify if I need to, so I hope you do too.

The full script can be found on my github repo here - [https://github.com/andyr8939/Powershell/blob/master/azure/Setup-AzureVMBackups.ps1](https://github.com/andyr8939/Powershell/blob/master/azure/Setup-AzureVMBackups.ps1)

Thanks!

Andy

