# Graded Lab Activity #7

**Monitoring an Azure virtual machine with Azure Monitor: alerts, action groups, alert processing rules and Log Analytics**

| | |
|---|---|
| **Name** | Hye Ran Yoo |
| **Student number** | 041145212 |
| **Course** | CST8912 – Cloud Solution Architecture |
| **Section** | 013 |
| **Date** | November 9, 2026 |

## What this lab does

I set up Azure Monitor for a virtual machine. When a VM in my resource group is deleted, the operations team gets an email. The lab scenario says that administrators must be notified of any significant infrastructure changes.

1. Use a template to provision an infrastructure (Task 1). The template creates a VM, a Log Analytics workspace and a data collection rule, so the VM sends its data to the workspace.
2. Create an alert (Task 2). The alert rule watches the activity log signal "Delete Virtual Machine".
3. Configure action group notifications (Task 3). The action group sends an email, and I add it to the alert rule.
4. Trigger an alert and confirm it is working (Task 4). I delete the VM and check the alert and the email.
5. Configure an alert processing rule (Task 5). The rule stops notifications during planned maintenance at night.
6. Use Azure Monitor log queries (Task 6). I query the VM data in Log Analytics with KQL.
7. Delete everything.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 25, "rankSpacing": 40, "padding": 6}, "themeVariables": {"fontSize": "15px"}}}%%
flowchart TB
    ME["<b>Me</b><br/>Azure portal"]
    subgraph RG["Resource group CST8912 · Canada Central"]
        VM["<b>az104-vm0</b> (Task 1)<br/>Windows Server 2022 · Standard_B2als_v2<br/>Azure Monitor Agent"]
        DCR["<b>az104-dcr11</b> (Task 1)<br/>Data collection rule<br/>performance data every 60 seconds"]
        LAW["<b>az104-law11-dmex6hh5j2mjw</b> (Task 1)<br/>Log Analytics workspace<br/>tables: Heartbeat, InsightsMetrics"]
        AR["<b>VM was deleted</b> (Task 2)<br/>Activity log alert rule<br/>signal: Delete Virtual Machine"]
        AG["<b>Alert the operations team</b> (Task 3)<br/>Action group · AlertOpsTeam<br/>Email"]
        APR["<b>Planned Maintenance</b> (Task 5)<br/>Alert processing rule · Suppress<br/>Nov 9 10:00 PM to Nov 10 7:00 AM"]
    end
    AL["<b>Activity log</b><br/>Delete Virtual Machine<br/>2:21 PM"]
    MAIL["<b>My email</b><br/>3 alert emails<br/>2:25 PM to 2:26 PM"]
    LOGS["<b>Monitor → Logs</b> (Task 6)<br/>Count heartbeats: 30<br/>CPU time chart"]
    VM -->|"heartbeat, CPU"| DCR
    DCR --> LAW
    LAW --> LOGS
    ME ==>|"Task 4: delete VM"| VM
    VM -.->|"delete event"| AL
    AL ==> AR
    AR ==>|"alert fired 2:25 PM"| AG
    AG ==> MAIL
    APR -.->|"no email during maintenance"| AG
    classDef data fill:#E3F0FC,stroke:#0078D4,stroke-width:2px,color:#0B2A4A
    classDef vm fill:#FFF1E0,stroke:#D9480F,stroke-width:2px,color:#4A1F00
    classDef alert fill:#E5F6EC,stroke:#2B8A3E,stroke-width:2px,color:#0E3B1B
    classDef quiet fill:#F3E8FF,stroke:#6F42C1,stroke-width:2px,color:#2E1A47
    classDef outside fill:#FFFFFF,stroke:#555555,stroke-width:2px,color:#222222
    class DCR,LAW,LOGS data
    class VM vm
    class AR,AG,MAIL alert
    class APR quiet
    class ME,AL outside
    style RG fill:#F7FBFF,stroke:#0078D4,stroke-dasharray:4 3,color:#0B2A4A
```

The blue path is the data path (Task 1 and Task 6). The green path is the alert path (Tasks 2, 3 and 4). The purple rule only stops the email at night (Task 5). All resources are in the resource group `CST8912` in Canada Central.

**Where I set each part.**

| Task | Portal page | What I set |
|---|---|---|
| 1 | Deploy a custom template | VM, network, storage account, Log Analytics workspace, data collection rule |
| 1 | Monitor → Insights → Virtual Machines | VM insights (it was already enabled by the template) |
| 2 | Monitor → Alerts → Create → Alert rule | Scope `CST8912`, signal "Delete Virtual Machine" |
| 3 | Alert rule → Actions → Create action group | Email to the operations team (my email) |
| 4 | Virtual machines → Delete | Deleted `az104-vm0` to fire the alert |
| 5 | Monitor → Alerts → Alert processing rules → Create | Suppress notifications from 10:00 PM to 7:00 AM |
| 6 | Monitor → Logs | KQL queries on `Heartbeat` and `InsightsMetrics` |
| Clean up | Resource groups → `CST8912` → Delete | Deleted all 11 resources |

## Changes from the instructions

| Instruction | What I did | Why |
|---|---|---|
| Download `\Allfiles\Lab11\az104-11-vm-template.json` | Used `az104-11-vm-template.json` from the Microsoft AZ-104 lab files on GitHub (Lab 11) | The file was not attached to the instructions. |
| VM size in the template: `Standard_D2s_v5`, `D2s_v6` or `D2s_v7` | `Standard_B2als_v2` (2 vCPU, 4 GiB) | The template allows only D2s sizes. In Canada Central, my DSv5 quota is 0 vCPU. `D2s_v7` is not in Canada Central. I checked this in Cloud Shell before I started. So I added `Standard_B2als_v2` and `Standard_B2ls_v2` to the allowed sizes, and I set `Standard_B2als_v2` as the default. The lab instructor said on September 18 that we can use any size that works in our region. |
| Step 1-11: "one virtual network with one virtual machine" | The resource group had 9 resources | The template now also creates a storage account (for boot diagnostics), a Log Analytics workspace, a data collection rule and the Azure Monitor Agent. |
| Steps 1-16 and 1-17: Enable VM insights | I did not click Enable | The template already turned on VM insights (data collection rule `az104-dcr11`). |
| Not in the instructions | Verified my email with a one-time code | Azure now sends a verification email to a new email in an action group. Without this step, the action group does not send emails. |
| Step 4-4: Apply force delete | Deleted without force delete | I deleted the VM from the Virtual machines list. This delete window did not have the "Apply force delete" option. The VM was deleted, and the alert still fired. |

Step numbers below follow the tasks of the instructions and count from 1 inside each task: for example, 2-5 is Task 2 ("Create an alert"), step 5.

## Task 1. Use a template to provision an infrastructure (steps 1-1 to 1-18)

**Before step 1-1. Check what my subscription can use.**
- I checked the VM sizes and quotas in Canada Central with Cloud Shell.
- `Standard_B2als_v2` has no restriction, and the Basv2 family quota is 0 of 10 vCPUs used.
- The DSv5 family quota is 0 of 0, so `Standard_D2s_v5` cannot be created.
- The Windows Server image `2022-datacenter-g2` is available.

![VM sizes and quota in Canada Central](screenshots/00-precheck-canadacentral-sizes-quota.png)

**Steps 1-3 to 1-6. Deploy a custom template → Build your own template in the editor → Load file.**

![Template loaded in the editor](screenshots/01a0-template-loaded.png)

**Before step 1-7. Change the allowed VM sizes in the editor.**
- `defaultValue` is `Standard_B2als_v2`. Two sizes are added to `allowedValues`. The reason is written in the `description`.

![VM size changed in the template](screenshots/01a-template-edit-vmsize.png)

**Steps 1-7 to 1-8. Save → fill in the Basics tab.**

| Setting | Value |
|---|---|
| Subscription | Azure for Students |
| Resource group | `CST8912` (new) |
| Region | Canada Central |
| Admin Username | `localadmin` |
| Admin Password | a complex password |
| Vm Size | `Standard_B2als_v2` |
| Other fields | default values |

![Custom deployment Basics, part 1](screenshots/01b-custom-deployment-basics-1.png)

![Custom deployment Basics, part 2](screenshots/01b-custom-deployment-basics-2.png)

**Step 1-9. Review + create → Create.**

![Review + create](screenshots/01c-review-create-values.png)

**Step 1-10. The deployment is complete.**
- Start time 1:45 PM. All 10 deployment items are OK.

![Deployment complete](screenshots/01d-deployment-complete.png)

**Step 1-11. Review the deployed resources.**
- The resource group `CST8912` has 9 resources: data collection rule, Log Analytics workspace, network interface, network security group, public IP address, virtual machine, OS disk, virtual network and storage account.

![Resources in CST8912](screenshots/01e-resource-group-resources.png)

**Steps 1-12 to 1-14. Monitor → review the insights, detection, triage and diagnosis tools.**

![Monitor overview](screenshots/01f-monitor-overview.png)

- On the Alerts page there were no alerts yet.

![Alerts before the test](screenshots/01f2-monitor-alerts-before.png)

**Step 1-15. VM Insights → View → Configure Insights.**

![VM insights get started](screenshots/01g0-vm-insights-get-started.png)

**Steps 1-16 to 1-18. Select the virtual machine.**
- `az104-vm0` is already **Monitored** and **Enabled** with the data collection rule `az104-dcr11`.

![VM insights enabled](screenshots/01g-vm-insights-enabled.png)

- The monitor settings show log-based metrics going to the workspace `az104-law11-dmex6hh5j2mjw`. I did not change anything here.

![VM monitor settings](screenshots/01g2-vm-monitor-settings.png)

## Task 2. Create an alert (steps 2-1 to 2-8)

**Steps 2-1 to 2-3. Monitor → Alerts → Create → Alert rule → Scope.**
- I selected the resource group `CST8912`, so the rule covers every VM in the group.

![Alert rule scope](screenshots/02a-alert-scope-rg.png)

**Steps 2-4 to 2-5. Condition → See all signals → Delete Virtual Machine (Virtual Machines).**
- The search shows 13 activity log signals. I selected the first one.

![Delete Virtual Machine signal](screenshots/02b-alert-signal-delete-vm.png)

**Steps 2-6 to 2-8. Alert logic.**
- Event level: All selected. Status: All selected. Event initiated by: all services and users.
- Condition preview: "Whenever the Activity Log has an event with Category='Administrative', Signal name='Delete Virtual Machine (Virtual Machines)'".

![Alert logic](screenshots/02c-alert-logic.png)

## Task 3. Configure action group notifications (steps 3-1 to 3-6)

**Steps 3-1 to 3-2. Next: Actions → Create action group → Basics.**

| Setting | Value |
|---|---|
| Subscription | Azure for Students |
| Resource group | `CST8912` |
| Region | Global |
| Action group name | `Alert the operations team` |
| Display name | `AlertOpsTeam` (12 characters, the limit) |

![Action group Basics](screenshots/03a-action-group-basics.png)

**Steps 3-3 to 3-4. Notifications → Email/SMS message/Push/Voice → Email.**
- Name: `VM was deleted`. I entered my school email.

![Email notification](screenshots/03b0-action-group-email-pane.png)

![Notifications tab](screenshots/03b-action-group-notifications.png)

**Review + create the action group → Create.**

![Action group review](screenshots/03b2-action-group-review.png)

- Back in the alert rule, the Actions tab shows the action group with 1 Email.

![Action group added to the rule](screenshots/03c-alert-rule-actions.png)

**Email verification (not in the instructions).**
- At 2:06 PM I got "Verify your email address" with a one-time password. I entered the code on the verification page.

![Verification email](screenshots/03g-email-verify-otp.png)

![Email verified](screenshots/03h-email-verified.png)

**Step 3-5. Details.**

| Setting | Value |
|---|---|
| Resource group | `CST8912` |
| Region | Global |
| Alert rule name | `VM was deleted` |
| Alert rule description | `A VM in your resource group was deleted` |
| Enable alert rule upon creation | checked |

![Alert rule Details](screenshots/03d-alert-rule-details.png)

**Step 3-6. Review + create → Create.**

![Alert rule review](screenshots/03e-alert-rule-review.png)

- At 2:11 PM the rule was in the list: target scope `CST8912`, signal type Activity log, severity 4 - Verbose, status Enabled.

![Alert rule list](screenshots/03f-alert-rule-list.png)

![Alert rule overview](screenshots/03f2-alert-rule-overview.png)

## Task 4. Trigger an alert and confirm it is working (steps 4-1 to 4-11)

**Steps 4-1 to 4-4. Virtual machines → check `az104-vm0` → Delete.**
- I waited about 10 minutes after the rule was enabled, as the note in the instructions says. Then I typed `delete` and deleted the VM at 2:21 PM.

![Delete the VM](screenshots/04a-vm-delete.png)

**Step 4-5. The VM is deleted.**

![No virtual machines](screenshots/04b-vm-deleted.png)

**Waiting for the alert.**
- At 2:24 PM there was still no alert. I checked the Activity log. The event "Delete Virtual Machine" was there with status Succeeded at 2:21:20 PM, so I waited.

![Activity log: Delete Virtual Machine](screenshots/04d1-activity-log-delete-vm.png)

**Steps 4-6 to 4-7. Alert email.**
- I got three emails "Important notice: Azure Monitor alert VM was deleted was activated" at 2:25 PM, 2:25 PM and 2:26 PM. Before that, I got "You're now in the AlertOpsTeam action group" at 2:10 PM.

![Alert emails in the inbox](screenshots/04c2-alert-email-inbox.png)

- The email shows the alert `VM was deleted` for `az104-vm0`. The time is in UTC: 19:21 UTC is 2:21 PM in Ottawa.

![Alert email](screenshots/04c-alert-email.png)

- Another email of the same alert has `"statusCode":"Accepted"` in Properties. Each email is for a different step of the same delete operation.

![Alert email for the Accepted status](screenshots/04c3-alert-email-accepted.png)

**Steps 4-8 to 4-10. Monitor → Alerts.**
- There are 3 alerts named `VM was deleted`, severity 4 - Verbose, affected resource `az104-vm0`, fired at 2:25 PM.

![Fired alerts](screenshots/04d-monitor-alerts-list.png)

**Step 4-11. Alert details.**
- "Why did this alert fire?": the event `Microsoft.Compute/virtualMachines/delete` occurred. Category Administrative.

![Alert details](screenshots/04e-alert-details.png)

## Task 5. Configure an alert processing rule (steps 5-1 to 5-7)

**Steps 5-1 to 5-2. Alert processing rules → Create → Scope.**

![Processing rule scope](screenshots/05a-processing-rule-scope.png)

![Scope applied](screenshots/05a2-processing-rule-scope-applied.png)

**Step 5-3. Rule settings → Suppress notifications.**
- The portal explains: "The alert will still fire, but the action groups won't be invoked so you won't receive any notifications when it fires."

![Suppress notifications](screenshots/05b-processing-rule-suppress.png)

**Steps 5-4 to 5-5. Scheduling.**

| Setting | Value |
|---|---|
| Apply the rule | At a specific time |
| Start | 11/9/2026, 10:00 PM |
| End | 11/10/2026, 7:00 AM |
| Time zone | (UTC-05:00) Eastern Time (US & Canada) |

![Scheduling section](screenshots/05c-processing-rule-scheduling.png)

**Step 5-6. Details.**

| Setting | Value |
|---|---|
| Resource group | `CST8912` |
| Rule name | `Planned Maintenance` |
| Description | `Suppress notifications during planned maintenance.` |

![Processing rule Details](screenshots/05d-processing-rule-details.png)

**Step 5-7. Review + create → Create.**

![Processing rule review](screenshots/05e-processing-rule-review.png)

![Planned Maintenance created](screenshots/05f-processing-rule-created.png)

## Task 6. Use Azure Monitor log queries (steps 6-1 to 6-9)

**Steps 6-1 to 6-3. Monitor → Logs → Select scope → `CST8912`.**

![Logs scope](screenshots/06a-logs-scope.png)

**Step 6-4. Queries → Virtual machines.**

![Virtual machines queries](screenshots/06a2-queries-virtual-machines.png)

**Steps 6-5 to 6-7. Count heartbeats → Run.**

![Run Count heartbeats](screenshots/06a3-count-heartbeats-run.png)

- The query uses the `Heartbeat` table:

```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| summarize count() by Computer
```

- Result: `az104-vm0` 30. The agent sends one heartbeat every minute, so the VM was running for about 30 minutes before I deleted it.

![Count heartbeats result](screenshots/06b-count-heartbeats.png)

**Step 6-8. CPU utilization chart.**

```kusto
InsightsMetrics
| where TimeGenerated > ago(1h)
| where Name == "UtilizationPercentage"
| summarize avg(Val) by bin(TimeGenerated, 5m), Computer //split up by computer
| render timechart
```

- The average CPU was about 7% at 1:50 PM, the highest at about 22% at 2:00 PM, 14% to 16% at 2:05 PM and 2:10 PM, and 0% at 2:15 PM. The data stops because I deleted the VM at 2:21 PM.

![CPU utilization chart](screenshots/06c-insightsmetrics-timechart.png)

- The chart showed UTC time first. I changed Display time to Local Time (UTC-05:00) so the times match Ottawa time in this report.

![Display time set to local time](screenshots/06c1-display-time-local.png)

**Step 6-9.** I did not run other queries.

## Clean up and document (step 6-10)

**Step 6-10. Clean up the resources and document all the steps in the lab report.**

- I deleted the resource group `CST8912`. The delete window listed 11 resources: the action group, data collection rule, Log Analytics workspace, network interface, network security group, public IP address, OS disk, virtual network, storage account, alert processing rule and alert rule.

![Delete resource group](screenshots/07a-rg-delete.png)

- After the deletion, only `NetworkWatcherRG` is left. It was there before the lab, so I did not touch it.

![Resource groups after deletion](screenshots/07b-rg-deleted.png)

## Findings and analysis

- **Two kinds of data.** The alert rule uses the Activity log. The Activity log is a record of what is done to Azure resources, for example "Delete Virtual Machine". The queries in Task 6 use the Log Analytics workspace. The Azure Monitor Agent in the VM sends this data (`Heartbeat`, `InsightsMetrics`). The data collection rule tells the agent what to send and where.
- **The alert is not instant.** The delete was at 2:21 PM. The alerts fired at 2:25 PM. The emails came at 2:25 PM and 2:26 PM. While I waited, I checked the Activity log first.
- **One delete, three alerts.** I left Event level and Status as All. So the rule fired once for each step of the delete. This is the "verbose alerts" in the instructions. I got three alerts and three emails.
- **Email verification.** Azure asked me to verify the action group email with a one-time code. Without this step, no alert email is sent.
- **Suppress keeps the alert.** The alert processing rule stops only the action group (the email) from 10:00 PM to 7:00 AM. The alert still fires and shows on the Alerts page.
- **Time zones.** The alert email shows UTC time. The Alerts page and the chart (after I changed Display time) show Ottawa time (UTC-05:00).
- **Cost.** The workspace uses the PerGB2018 pricing tier. This means I pay for the amount of data that comes in. The data collection rule sends only performance data, every 60 seconds. I deleted the resource group right after the lab.
