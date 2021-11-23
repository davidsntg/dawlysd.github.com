---
layout: post
title:  "Azure VMs - Automatic patching with tags!"
author: davidsantiago
categories: [ azure, network ]
image: assets/images/update-management-big-picture.png
featured: true
---

The intent of this article is to explain how to daily or weekly patch Azure Virtual Machines with tags. To achieve that, I will rely on Azure Update Management and several Powershell Runbooks.

This article has been co-written with my friend [Vincent Misson](https://blog.cloud63.fr/).

# Azure Update Management

[Azure Update Management](https://docs.microsoft.com/en-us/azure/automation/update-management/overview) manages operating system updates for Windows and Linux Azure VMs or VMs in on-premise environments and even in other cloud environments.

To enable patching with Azure Updatement Management, we usually:

1. Create a Log Analytics Workspace
2. Create an Automation account and link it to the Log Analytics Workspace
3. [Enable Update Management from the Automation Account](https://docs.microsoft.com/en-us/azure/automation/update-management/enable-from-automation-account)
4. Install [Log Analytics agent](https://docs.microsoft.com/en-us/azure/azure-monitor/agents/log-analytics-agent) on VMs and specify the Workspace ID
5. [Schedule Update Deployment for VMs or Groups](https://docs.microsoft.com/en-us/azure/automation/update-management/deploy-updates)

Management at scale through the Portal can be complicated: who should and must perform operations on the automation account? How to prevent a user from creating/updating a schedule update deployment of a machine that does not belong to him/her? ... This is not really manageable at scale.

The bellow model simplifies all this management. Rather than "Schedule Update Deployment" in the Update Management tab of the Automation Account, we will **simply tag the machines we want to patch with a specific syntax**.

Automation Account Runbooks will take care of the update deployment schedule configuration 🔥. Let's see how it works overall.


# Big Picture

Assuming the following architecture:

![BigPicture]({{ site.baseurl }}/assets/images/update-management-big-picture.png)

There is :
* The automation Account which is linked to the Log analytics workspace. Update Management solution has been installed
* VMs in different subscriptions and resource groups with a `POLICY_UPDATE` tag

`POLICY_UPDATE` is a key tag that must be added on VMs we want to patch. An Automation Account Runbook will regularly list all the VMs with this key tag to configure the update deployment schedules.

## `POLICY_UPDATE` Tag!

The tag value must use the following syntax: `Day(s)OfWeek;startTime;rebootPolicy;excludedPackages;reportingMail`

Examples:
* VM1 - `POLICY_UPDATE=Friday;10:00 PM;Never;*java*;` VM1 will be patched every Friday, at 10:00 PM. Even if updates require reboot, the VM will not be rebooted. Packages containing `java` string will be excluded.
* VM2 - `POLICY_UPDATE=Tuesday,Sunday;08:00 AM;IfRequired;;TeamA@abc.com` VM2 will be patched every Tuesday and Sunday, at 08:00 AM. The VM will be rebooted only if a patch needs the machine to be reboot to be taken into account. No excluded packages. When patching is done, TeamA@abc.com will receive the list of updated packages by mail.
* VM3 - `POLICY_UPDATE=Sunday;07:00 PM;Always;;` VM3 will be patched every Synday at 07:00 PM. The VM will be rebooted after applying patches, even if it is not required. No excluded packages.
* VM4 - `POLICY_UPDATE=Monday;01:00 PM;Always;*java*,*oracle*;TeamB@abc.com` VM4 will be patched every Monday at 01:00 PM. The VM will be rebooted after applying patches. Packages containing `java` or `oracle` string will be excluded. When patching is done, TeamB@abc.com will receive the list of updated packages by mail.

`rebootPolicy` options are:
- Always
- Never
- IfRequired

Regarding `Day(s)OfWeek` and `excludedPackages`, if you want to put several values, they must be **comma separated**.

`Day(s)OfWeek`, `startTime` and `rebootPolicy` are mandatory. `excludedPackages` and `reportingMail` are optional parameters.

### Wrap-up

Here is the syntax to follow for the `POLICY_UPDATE` tag:
![POLICY_UPDATE_SYNTAX]({{ site.baseurl }}/assets/images/update-management-tag-syntax.png)

## Runbooks

All this automation is possible thanks to a set of five Runbooks (deployed in the Automation Account):
* **UM-ScheduleUpdatesWithVmsTags**: Must be scheduled (at least) daily. Searches for all machines with the `POLICY_UPDATE` tag and configures the Update Management schedules.
* **UM-PreTasks**: Triggered before patching, it can perform several optional actions like OS disk snapshot, start VM if stopped, etc..
* **UM-PostTasks**: Triggered after patching, it can perform several optional actions like stop VM if it was started, send patching email report, etc..
* **UM-CleanUp-Snapshots**: Must be scheduled daily, to delete snapshots that are X days older.
* **UM-CleanUp-Schedules**: Must be schedule (at least) daily. It removes Update Management schedules for VM machines that not longer have the `POLICY_UPDATE` tag

### Supported features

Runbooks allows you to:
* Patch **Azure Virtual Machines** and **Azure Arc Servers** with [Supported OS](https://docs.microsoft.com/en-us/azure/automation/update-management/operating-system-requirements#supported-operating-systems)
* Patch in a multi-subscriptions context: the system-assigned managed identity must have **Contributor** role assigned on each subscription.
* Perform several pre and post patching tasks:
  * Pre scripts, before patching:
    * [OPTIONAL] Snapshot VM OS disk 
    * [OPTIONAL] Start VM if it is stopped 
  * Post scripts, after patching:
    * [OPTIONAL] Shutdown VM if it was started by pre-script.
    * [OPTIONAL] Send a patching report email
* Support Azure Arc Servers
  * Pre-scripts and post scripts **are not supported for Azure Arc Servers**

Example of patching report email received:
![Reporting_Mail]({{ site.baseurl }}/assets/images/update-management-reporting-mail.png)

## Give it a try !

A quickstart Bicep script is available on **[GitHub](https://github.com/dawlysd/azure-update-management-with-tags)**.

Screenshot of deployed resources:
![Infrastructure]({{ site.baseurl }}/assets/images/update-management-quickstart-infrastructure.png)

### Unsupported scenarios

Below Azure Resource cannot be patched by Azure Update Management:
* [VMSS](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview): Leverage [automatic OS image upgrade](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-automatic-upgrade) instead.
* [AVD](https://docs.microsoft.com/en-us/azure/virtual-desktop/): Leverage [automatic updates](https://docs.microsoft.com/en-us/azure/virtual-desktop/configure-automatic-updates) instead.
* [AKS](https://docs.microsoft.com/en-us/azure/aks/): Leverage [auto-upgrade channel](https://docs.microsoft.com/en-US/azure/aks/upgrade-cluster#set-auto-upgrade-channel) and [planned-maintenance](https://docs.microsoft.com/en-us/azure/aks/planned-maintenance) instead.

### Limitations

* VMs must have a unique name. It is not possible to have several VMs with the same name using `POLICY_UPDATE` tag.
* A VM must be part of a single schedule. If it is not the case, *UM-CleanUp-Schedules* has side effects.
* When the `POLICY_UPDATE` tag is applied, the first patching can be done the day after the execution of *UM-ScheduleUpdatesWithVmsTags*.

# Conclusion

Provided runbooks could be improved to enhance modularity. The issue is that there is as much patching policy as company and providing scripts that fit all use cases is impossible.

Additional features can be developed (feel free to **contribute** on [GitHub repo](https://github.com/dawlysd/azure-update-management-with-tags)) as creating ITSM ticket before patching and update the ticket with updated packages after the patching.

Azure Tags are powerfull, especially for automation. We could imagine doing the same thing for many day-to-day operations as backup, monitoring and so on.

Enjoy and give us feedbacks 😁!