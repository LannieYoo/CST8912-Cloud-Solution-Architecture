# Graded Lab Activity #4

**Copying data from Azure SQL Database to Blob Storage with Azure Data Factory**

| | |
|---|---|
| **Name** | Hye Ran Yoo |
| **Student number** | 041145212 |
| **Course** | CST8912 – Cloud Solution Architecture |
| **Section** | 013 |
| **Date** | October 7, 2026 |

## What this lab does

I copy one table from an Azure SQL Database into a CSV file in Azure Blob Storage. Azure Data Factory does the copy. Before the copy, I check the table with two queries. All resources are in one resource group, and I delete the group at the end.

```mermaid
flowchart LR
    subgraph RG["Resource group cst8912-demo · Canada Central · deleted in Step 22"]
        direction LR
        SQL["<b>FROM</b><br/>Azure SQL Database<br/>db8912<br/>table SalesLT.Product<br/><i>Steps 1–9</i>"]
        ADF["<b>COPY</b><br/>Azure Data Factory<br/>demodb8912<br/>Copy Data tool<br/><i>Steps 12–21</i>"]
        BLOB["<b>TO</b><br/>Blob Storage<br/>demo8912<br/>productdata8912<br/>product.csv<br/><i>Steps 10–11</i>"]
        SQL ==>|"read the table"| ADF ==>|"write CSV, 295 rows"| BLOB
    end
    classDef from fill:#E3F0FC,stroke:#0078D4,stroke-width:3px,color:#0B2A4A
    classDef copy fill:#FFF1E0,stroke:#D9480F,stroke-width:3px,color:#4A1F00
    classDef to fill:#E5F6EC,stroke:#2B8A3E,stroke-width:3px,color:#0E3B1B
    class SQL from
    class ADF copy
    class BLOB to
    style RG fill:#FAFAFA,stroke:#9E9E9E,stroke-dasharray:6 4,color:#555555
    linkStyle 0,1 stroke:#333333,stroke-width:3px
```

## 1. Create Azure SQL Database (Steps 1–6)

**Step 1. Configure Azure SQL database for Canada central region under your resource group cst8912-demo, choose single database under sql databases in sql deployment option.**
- I opened SQL databases in the portal and clicked Create. This makes a single database.
- The resource group `cst8912-demo` was deleted at the end of Lab 3, so I created it again.

**Step 2. Enter the following values in create database page and keep other properties with their default settings.**
- Subscription: Azure for Students. Resource group: `cst8912-demo` (new). Database name: `db8912`.
- Server: Create new. The name `db8912demo` was already in use, so I used `db8912demohry`. Location: Canada Central. Authentication: SQL authentication. Server admin login: `db8912_hyeran`. Password: the one in the instructions.
- SQL elastic pool: No. Workload environment: Development. Compute + storage: not changed (General Purpose - Serverless, 1 vCore, 32 GB). Backup storage redundancy: Locally-redundant backup storage.

![Server name already in use](screenshots/01a-sql-server-name-taken.png)

![Create SQL Database Server](screenshots/01b-sql-server-new.png)

![Basics tab](screenshots/01c-sql-basics.png)

**Step 3. Select Next: Networking >, and on the Networking page, in the Network connectivity section, select Public endpoint. Then select Yes for both options in the Firewall rules section to allow access to your database server from Azure services and your current client IP address.**
- Connectivity method: Public endpoint. Allow Azure services and resources to access this server: Yes. Add current client IP address: Yes.

![Networking tab](screenshots/02-sql-networking.png)

**Step 4. Select Next: Security > and set the Enable Microsoft Defender for SQL option to Not now.**
- Enable Microsoft Defender for SQL: Not now.

![Security tab](screenshots/03-sql-security.png)

**Step 5. Select Next: Additional Settings > and on the Additional settings tab, set the Use existing data option to Sample.**
- Use existing data: Sample. The portal creates the AdventureWorksLT sample database.

![Additional settings tab](screenshots/04-sql-additional-sample.png)

**Step 6. Select Review + Create, and then select Create to create your Azure SQL database.**
- The review page shows all the settings from Steps 2 to 5. Then I clicked Create.

![Review + create, Basics](screenshots/05a-sql-review-create.png)

![Review + create, Networking, Security and Additional settings](screenshots/05b-sql-review-create-2.png)

**Result.**
- The database `db8912` is Online on the server `db8912demohry.database.windows.net` in Canada Central.

![SQL database overview](screenshots/05c-sql-overview.png)

## 2. Query the database (Steps 7–9)

**Step 7. In the pane on the left side of the page, select Query editor (preview), and then sign in using the administrator login and password you specified for your server.**
- I opened Query editor and signed in with SQL server authentication as `db8912_hyeran`. There was no client IP error, because my IP was added in Step 3.

![Query editor after sign in](screenshots/06a-query-editor-signed-in.png)

**Step 8. Expand the Tables folder to see the tables in the database.**
- My Query editor shows the tables under the schema name. Under `SalesLT` → Tables I can see Address, Customer, CustomerAddress, Product, ProductCategory, ProductDescription, ProductModel, ProductModelProductDescription, SalesOrderDetail and SalesOrderHeader.

![Tables expanded](screenshots/06b-tables-expanded.png)

**Step 9. In the query 1 pane, try executing the following queries, select run to execute the query.**
- Query 1 returned 295 rows with 4 columns.

```sql
SELECT ProductID, Name, ListPrice, ProductCategoryID
FROM SalesLT.Product;
```

![Query 1 result](screenshots/07a-query1-result.png)

- Query 2 joins Product with ProductCategory. It also returned 295 rows, and the Category column shows the category name, for example Mountain Bikes.

```sql
SELECT p.ProductID, p.Name AS ProductName,
       c.Name AS Category, p.ListPrice
FROM SalesLT.Product AS p
JOIN [SalesLT].[ProductCategory] AS c
    ON p.ProductCategoryID = c.ProductCategoryID;
```

![Query 2 result](screenshots/07b-query2-result.png)

## 3. Create Storage Account and Container (Steps 10–11)

**Step 10. Create a Azure storage account with the following settings, keeping the other advanced, networking, data protection, encryption settings default.**
- Resource group `cst8912-demo`, storage account name `demo8912`, region Canada Central, performance Standard, redundancy Locally-redundant storage (LRS).
- Primary service: Azure Blob Storage (the file will be a blob). The other tabs were left as default.

![Storage account Basics](screenshots/08b-storage-basics.png)

![Storage account review, part 1](screenshots/08c-storage-review-1.png)

![Storage account review, part 2](screenshots/08d-storage-review-2.png)

**Result.**
- `demo8912` is a StorageV2 account, Standard, LRS, in canadacentral.

![Storage account overview](screenshots/08e-storage-overview.png)

**Step 11. Create a container "productdata8912" in storage account.**
- Data storage → Containers → + Add container → `productdata8912`. The access level is Private.

![Container list](screenshots/09-container-created.png)

## 4. Create Data Factory (Steps 12–13)

**Step 12. Create a new resource in your resource group from the azure portal, search for azure data factory with the following configuration, keeping git configuration, networking, advanced as default.**
- Subscription Azure for Students, resource group `cst8912-demo`, name `demodb8912`, region Canada Central, version V2. Git configuration, Networking and Advanced were left as default.

![Data Factory Basics](screenshots/10a-adf-basics.png)

![Data Factory review](screenshots/10b-adf-review-create.png)

**Step 13. Once created launch azure data factory studio.**
- Azure Data Factory Studio opened in a new tab for `demodb8912`.

## 5. Copy data with Data Factory (Steps 14–21)

**Step 14. On home page, choose option to ingest data.**
- On the Studio home page I clicked Ingest. This opens the Copy Data tool.

![Studio home page](screenshots/12-studio-home-ingest.png)

**Step 15. Choose task type, built in copy task, and task cadence "run once now".**
- Task type: Built-in copy task. Task cadence: Run once now.

![Copy Data tool properties](screenshots/13-copy-properties.png)

**Step 16. In source type choose azure sql database from the dropdown, and connection, choose new connection with the following configuration and test connection.**
- Source type: Azure SQL Database. New connection `AzureSqlDatabase1`: server `db8912demohry`, database `db8912`, SQL authentication, user name `db8912_hyeran` and the password from Step 2.
- Test connection: Connection successful.

![Source connection test](screenshots/14a-source-connection-ok.png)

**Step 17. Select source table as "SalesLT.Product" from the dropdown, click next and you can preview data.**
- I checked `SalesLT.Product` and clicked Preview data. The preview shows the same rows as Query 1.

![Source table selected](screenshots/14b-source-table.png)

![Preview data](screenshots/14c-source-preview.png)

**Step 18. Click next, choose destination type, select "Azure Blob Storage" from dropdown.**
- Destination type: Azure Blob Storage.

**Step 19. Create new connection and test connection to storage account, choose the folder path and enter file name.**
- New connection `AzureBlobStorage1`: authentication type Account key, storage account `demo8912`. Test connection: Connection successful.
- Folder path: `productdata8912/`. File name: `product.csv`.

![Destination connection test](screenshots/15a-dest-connection-ok.png)

![Folder path and file name](screenshots/15b-dest-folder-file.png)

**Step 20. Choose the configuration.**
- File format DelimitedText, column delimiter Comma, row delimiter Default, Add header to file checked, no compression. The Settings page was left as default.

![File format settings](screenshots/16-file-format.png)

**Step 21. Review and finish this pipeline and check the storage account container to see the product csv file copied from the database to storage account.**
- The summary shows the copy from Azure SQL Database (`SalesLT.Product`) to Azure Blob Storage (`productdata8912` / `product.csv`).

![Review and finish summary](screenshots/17a-review.png)

- The deployment created the datasets and the pipeline and ran the pipeline. All steps succeeded.

![Deployment complete](screenshots/17b-deployment-complete.png)

- In Monitor, the pipeline run `CopyPipeline_xy4` succeeded. It took 24 seconds.

![Pipeline run succeeded](screenshots/17c-monitor-succeeded.png)

- The file `product.csv` (1.3 MiB) is in the container `productdata8912`.

![product.csv in the container](screenshots/18a-container-product-csv.png)

- The Edit tab shows the header line and the product rows, starting with product 680.

![product.csv content](screenshots/18b-product-csv-content.png)

- I downloaded the file and opened it in Excel. It has 17 columns and 295 rows, the same as the table.

![product.csv in Excel](screenshots/18c-product-csv-excel.png)

## 6. Clean up and document (Step 22)

**Step 22. After demo delete all the resources created during this lab and create a lab report documenting all the steps performed in the lab along with the screenshots.**
- I deleted the resource group `cst8912-demo`. The confirmation panel listed 5 resources: the SQL server, the databases `db8912` and `master`, the storage account and the data factory.

![Delete resource group confirmation](screenshots/19a-rg-delete-confirm.png)

- After the deletion, only `NetworkWatcherRG` is left. Azure created that group by itself, so I did not touch it.

![Resource groups after deletion](screenshots/19b-rg-deleted.png)

## 7. Findings and analysis

- **Azure SQL Database is a PaaS service.** I only chose the database name, the server name, the login and the firewall rules. Azure manages the server, the patches and the backups.
- **The two firewall settings have two jobs.** "Add current client IP address" lets my computer use the Query editor. "Allow Azure services and resources" lets Data Factory connect to the database. Without it, the test connection in Step 16 would fail.
- **The sample data made the lab possible.** The Sample option created the `SalesLT` tables. Both queries returned 295 rows. The JOIN query shows the category name instead of the category ID.
- **Data Factory copied the table with no code.** The Copy Data tool made two connections, two datasets and one pipeline, and ran it once. The run took 24 seconds. The CSV file has a header line and the same 295 rows as the table. The file is 1.3 MiB because the table also has a product photo column.
- **Clean up.** The Development workload uses a serverless database that pauses after 1 hour, so the cost is low. Deleting one resource group removed the SQL server, the databases, the storage account and the data factory in one step.
