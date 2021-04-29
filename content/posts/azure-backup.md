---
title: "Azure Backups for VMs and SQL in a VM"
date: 2021-04-29T13:51:26+12:00
draft: true
---

Backups are a necessity but it still amazes me the amount of companies I have worked at where you find backups are just not done, or just as bad, are done but are never checked!

> A backup is only a backup when you have done a successful restore

There are multiple ways to do backups depending where your data lives.  Veeam, BackupExec, etc, but in you are running your infrastructure in Azure and you are using Virtual Machine then [Azure Backup](https://docs.microsoft.com/en-us/azure/backup/backup-overview) is an amazing tool for this.

You can setup Azure via the Portal, and this is great if you are doing this once or twice, but what if you are doing this at scale for multiple accounts?   **Powershell**....  I created a script one day when I realized I was going to be setting up backups more than once, and I have used this script now at least 80 times, saving me so much time.  Its pretty much set a few variables up front, execute the script, get a coffee, come back 20 minutes later and you have a full backup solution.   The extra benefit of this is that a lot of the clients I have worked at have SQL Server running inside a Windows VM still, alongside the usual Azure SQL.  Azure Backup if you tell it too, will backup the SQL Server in a VM and do per database level backups which you can easily restore from which is super handy.

You can find the script in [my github repo here](https://github.com/andyr8939/Powershell/blob/master/azure/Setup-AzureVMBackups.ps1) but I'll give you the overview of it below.

## Requirements

You are going to need a few things first.
1. An Azure Subscription
2. A Resource Group
3. Some Virtual Machines running in the resource group, windows or linux, it doesn't matter.
4. Optional but if you have SQL Server on a VM, this script also covers that.

## Initial Variable