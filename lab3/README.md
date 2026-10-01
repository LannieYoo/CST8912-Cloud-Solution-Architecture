# Graded Lab Activity #3

**Azure Storage Account & SAS Tokens**

| | |
|---|---|
| **Name** | Hye Ran Yoo |
| **Student number** | 041145212 |
| **Course** | CST8912 – Cloud Solution Architecture |
| **Section** | 013 |
| **Date** | October 1, 2026 |
| **Lab title** | Graded Lab Activity #3 |

In this lab I created an Azure storage account, changed its redundancy and access tier, uploaded a blob, tested private access and a shared access signature (SAS), created a lifecycle management rule, and deleted all resources.

**Lab flow**

```text
1. Create storage account  labtest8912hry  (Canada Central, GRS)
        |
2. Change redundancy  GRS -> LRS      Change default access tier  Hot -> Cool
        |
3. Create container  labtestcontainer8912  ->  upload  sampletest8912/addresses.csv  as Hot
        |
4. Open blob URL in a browser                ->  access denied
        |
5. Open blob URL + SAS token (read only)     ->  file downloaded
        |
6. Lifecycle rule  myrule8912:  not modified for more than 15 days  ->  move to Cool
        |
7. Delete resource group  CST8912-demo
```

## 1. Create Storage Account (Steps 1–5)

**Step 1. Create Storage Account- Create a storage account named labtest8912 under your student subscription.**
- `labtest8912` was already taken, so I used `labtest8912hry` (my initials added).

![Storage account name already taken](screenshots/01a-name-taken.png)

**Steps 2–5. Use the resource group CST8912-demo. Select Canada Central as the region. Choose Geo-redundant storage (GRS). Keep networking and data protection options as default.**
- New resource group `CST8912-demo`, Canada Central, GRS with the read-access option unchecked; the other tabs left as default.

![Basics tab before creation](screenshots/01b-basics-filled.png)

**Result.**
- Replication: Geo-redundant storage (GRS) – primary Canada Central, secondary Canada East. The same page shows the default networking and data protection values (public network access from all networks, soft delete 7 days).

![Storage account overview](screenshots/01c-overview.png)

## 2. Modify Redundancy and Access Tier (Steps 6–9)

**Steps 6–7. Modify Redundancy and Access Tier. Go to your storage account → Data Management → Redundancy.**
- Before the change: GRS with two locations, Canada Central (primary) and Canada East (secondary).

![Redundancy GRS](screenshots/02a-redundancy-grs.png)

**Step 8. Change redundancy from Geo-redundant (GRS) to Local redundant (LRS).**
- After saving, only Canada Central is left.

![Redundancy LRS](screenshots/02b-redundancy-lrs.png)

**Step 9. Under Configuration, set Blob access tier to Cool, then save.**
- The default tier was Hot; changed to Cool and saved.

![Configuration Cool](screenshots/02c-config-cool.png)

## 3. Create Container and Upload Blob (Steps 10–14)

**Steps 10–11. Create Container and Upload Blob. Under Data Storage, click Containers → create a container named labtestcontainer8912.**
- Anonymous access level Private (the only option, because anonymous access is disabled on the storage account).

![Container list](screenshots/03a-container-created.png)

**Steps 12–14. Upload a blob into folder sampletest8912. Change Advanced Settings → Access Tier to Hot. Use the sample files link provided (ask instructor if not available).**
- `addresses.csv` from the sample files link; Advanced → Access tier Hot, Upload to folder `sampletest8912`.

![Upload blob advanced settings](screenshots/03b-upload-advanced.png)

**Result.**
- `sampletest8912/addresses.csv` (324 B) is in the Hot tier, not the account default Cool.

![Blob in Hot tier](screenshots/03c-blob-hot.png)

## 4. Test Private Access (Steps 15–19)

**Steps 15–18. Test Private Access. Click the uploaded file in the container. Copy the Blob URL. Open a private/incognito browser window, paste the URL.**
- Blob URL copied from the blob overview and opened in a browser.

**Step 19. Verify that it does not work (public access is private → resource not found).**
- It did not open: `PublicAccessNotPermitted` – "Public access is not permitted on this storage account." The message differs from "resource not found" because anonymous access is disabled at the account level, but the result is the same: no access without authorization.

![Private access denied](screenshots/04-private-access-denied.png)

## 5. Generate Shared Access Signature (SAS) (Steps 20–24)

**Steps 20–21. Generate Shared Access Signature (SAS). On the file blade, click Generate SAS.**
- Default settings: Account key (Key 1), permissions Read, start and expiry about 8 hours, HTTPS only → Generate SAS token and URL.

![Generate SAS](screenshots/05a-generate-sas.png)

**Steps 22–23. Copy the SAS Token URL. Paste it into the private browser window.**
- The URL has the SAS token after `?` (`sp=r`, `st`, `se`, `spr=https`, `sr=b`, `sig`).

![SAS URL in the address bar](screenshots/05c-sas-url-in-address-bar.png)

**Step 24. Verify that you can now access the file.**
- The file was downloaded (`addresses (1).csv`, 324 B); the same window still shows the error for the URL without the token.

![File downloaded with SAS](screenshots/05b-sas-download-done.png)

## 6. Create Lifecycle Management Rule (Steps 25–28)

**Steps 25–27. Create Lifecycle Management Rule. On the container blade, under Data Management → Lifecycle Management, create a new rule: Rule Name: myrule8912. Scope: Limit blobs with filters -- Blob type/subtype: Default.**
- The menu is under Data management of the storage account; rule name `myrule8912`, Limit blobs with filters, Block blobs, Base blobs.

![Rule details](screenshots/06a-rule-details.png)

**Step 28. Condition: Base blobs last modified more than 15 days ago → Move to Cool storage.**
- If last modified more than 15 days ago, then move to cool storage.

![Rule condition](screenshots/06b-rule-base-blobs.png)

**Step 27 – filter for "Limit blobs with filters".**
- Blob prefix `labtestcontainer8912/`. The instructions say "on the container blade", so I limited the rule to this container.

![Rule filter](screenshots/06c-rule-filter-set.png)

**Result.**
- `myrule8912` is Enabled.

![Rule list](screenshots/06d-rule-list.png)

## 7. Clean Up & Document (Steps 29–30)

**Step 29. Clean Up & Document--Delete all resources created during this lab.**
- Deleted the resource group `CST8912-demo`; the confirmation panel listed one resource, the storage account `labtest8912hry`.

![Delete resource group confirmation](screenshots/07a-rg-delete-confirm.png)

**Result.**
- `CST8912-demo` is gone. `NetworkWatcherRG` was created automatically by Azure, so I did not touch it.

![Resource groups after deletion](screenshots/07b-rg-deleted.png)

**Step 30. Submit a lab report with screenshots for each step.**
- This report.

## 8. Findings and analysis

- **Redundancy.** With GRS, Azure kept a second copy of the data in a secondary region, Canada East. After the change to LRS, only Canada Central was left. GRS protects the data from a regional outage; LRS is cheaper. For test data like this lab, LRS is enough. The change from GRS to LRS was one setting in the portal.
- **Access tier.** The account default tier was Cool, but the blob was uploaded as Hot, so the blob stayed in the Hot tier. The tier set on the blob overrides the account default. Hot is for data that is read often; Cool is for data that is read less often and costs less to store.
- **Private access.** The blob URL alone did not open the file. The storage account has `Allow Blob anonymous access` disabled, so the error was `PublicAccessNotPermitted`. A private container does not give access to anyone who only knows the URL.
- **SAS.** The same file opened with the SAS URL. The SAS gave read-only access to one blob for a limited time, and it was signed with the account key. I did not need to share the account key or change the container to public.
- **Lifecycle management.** The rule moves blobs that were not modified for more than 15 days to the Cool tier automatically. The portal notes that a new rule can take up to 24 hours to take effect. The blob was uploaded on the same day, so the rule had nothing to move during the lab. This kind of rule lowers storage cost without manual work.
- **Clean up.** Deleting the resource group removed the storage account, the container, the blob and the lifecycle rule in one step.
