# Ingest-Realtime-Data-with-Azure-Stream-Analytics-and-Azure-Synapse-Analytics

This document outlines the steps to use Azure Stream Analytics to process a stream of simulated sales order data from Azure Event Hubs and ingest it into Azure Synapse Analytics in two ways: directly into a dedicated SQL pool and aggregated into Azure Data Lake Storage Gen2.

## Prerequisites

* An Azure subscription with administrative-level access.

## Steps

### 1. Provision Azure Resources

Provision an Azure Synapse Analytics workspace with a dedicated SQL pool and Azure Event Hubs using the provided PowerShell script and ARM template.

1.  Sign in to the [Azure portal](https://portal.azure.com).
2.  Open **Cloud Shell** by clicking the **[\>_]** button. Select **PowerShell** if prompted.
3.  Clone the repository:
    ```powershell
    rm -r dp-203 -f
    git clone [https://github.com/MicrosoftLearning/dp-203-azure-data-engineer](https://github.com/MicrosoftLearning/dp-203-azure-data-engineer) dp-203
    ```
4.  Navigate to the lab directory and run the setup script:
    ```powershell
    cd dp-203/Allfiles/labs/18
    ./setup.ps1
    ```
5.  If prompted, choose your subscription and enter a password for your Azure Synapse SQL pool. **Remember this password!**
6.  Wait for the script to complete (approximately 15 minutes).

### 2. Ingest Streaming Data into a Dedicated SQL Pool

Configure a Stream Analytics job to ingest raw order data into a Synapse SQL pool table.

#### 2.1. View the Streaming Source and Database Table

1.  After the script completes, in the Azure portal, navigate to the `dp203-xxxxxxx` resource group and note the Azure Synapse workspace (`synapsexxxxxxx`), the dedicated SQL pool (`sqlxxxxxxx`), and the Event Hubs namespace (`eventsxxxxxxxx`).
2.  Open **Synapse Studio** from your Synapse workspace's Overview page.
3.  On the **Manage** page, start the `sqlxxxxxxx` dedicated SQL pool.
4.  While the pool starts, open Cloud Shell and run the order client:
    ```powershell
    node ~/dp-203/Allfiles/labs/18/orderclient
    ```
    Observe the simulated order data.
5.  Once the client finishes, in Synapse Studio, on the **Data** page, expand your `sqlxxxxxxx` SQL pool and its tables. Select `dbo.FactOrder` and view its schema (it should have `OrderDateTime`, `ProductID`, and `Quantity` columns but no data initially).

#### 2.2. Create an Azure Stream Analytics Job to Ingest Order Data

1.  In the Azure portal, create a new **Stream Analytics job** named `ingest-orders` in the same region as your Synapse workspace.
2.  Configure the job's **Storage account** to use the `datalakexxxxxxx` storage account with Connection string authentication.

#### 2.3. Create an Input for the Event Data Stream

1.  In the `ingest-orders` job, add an **Event Hub** input named `orders` connected to your `eventsxxxxxxxx` Event Hub and the `$Default` consumer group.
2.  Use **Create system assigned managed identity** for authentication.
3.  Set the **Event serialization format** to `JSON` and **Encoding** to `UTF-8`. Save the input.

#### 2.4. Create an Output for the SQL Table

1.  Add an **Azure Synapse Analytics** output named `FactOrder` connected to your `sqlxxxxxxx` database.
2.  Use **SQL server authentication** with the username `SQLUser` and the password you set during setup.
3.  Set the **Table** to `FactOrder`. Save the output.

#### 2.5. Create a Query to Ingest the Event Stream

1.  In the `ingest-orders` job, go to the **Query** page.
2.  Replace the default query with:
    ```sql
    SELECT
        EventProcessedUtcTime AS OrderDateTime,
        ProductID,
        Quantity
    INTO
        [FactOrder]
    FROM
        [orders]
    ```
3.  Save the query.

#### 2.6. Run the Streaming Job to Ingest Order Data

1.  Start the `ingest-orders` Stream Analytics job.
2.  Run the order client again in Cloud Shell:
    ```powershell
    node ~/dp-203/Allfiles/labs/18/orderclient
    ```
3.  In Synapse Studio, re-run the "Select TOP 100 rows" query on `dbo.FactOrder` to verify that data is now being ingested.
4.  Pause the `sqlxxxxxxx` dedicated SQL pool in Synapse Studio.
5.  Stop the `ingest-orders` Stream Analytics job in the Azure portal.

### 3. Summarize Streaming Data in a Data Lake

Configure a second Stream Analytics job to aggregate order data and write it to Azure Data Lake Storage Gen2.

#### 3.1. Create an Azure Stream Analytics Job to Aggregate Order Data

1.  In the Azure portal, create a new **Stream Analytics job** named `aggregate-orders` in the same region.
2.  Configure the job's **Storage account** to use the `datalakexxxxxxx` storage account with Connection string authentication.

#### 3.2. Create an Input for the Raw Order Data

1.  In the `aggregate-orders` job, add an **Event Hub** input named `orders` connected to your `eventsxxxxxxxx` Event Hub and the `$Default` consumer group.
2.  Use **Create system assigned managed identity** for authentication.
3.  Set the **Event serialization format** to `JSON` and **Encoding** to `UTF-8`. Save the input.

#### 3.3. Create an Output for the Data Lake Store

1.  Add a **Blob storage/ADLS Gen2** output named `datalake` connected to your `datalakexxxxxxx` storage account and the `files` container.
2.  Use **Connection string** authentication.
3.  Set the **Event serialization format** to `CSV - Comma (, )` and **Encoding** to `UTF-8`.
4.  Set **Write mode** to `Append as results arrive` and **Path pattern** to `{date}` with **Date format** `YYYY/MM/DD`. Save the output.

#### 3.4. Create a Query to Aggregate the Event Data

1.  In the `aggregate-orders` job, go to the **Query** page.
2.  Replace the default query with:
    ```sql
    SELECT
        DateAdd(second,-5,System.TimeStamp) AS StartTime,
        System.TimeStamp AS EndTime,
        ProductID,
        SUM(Quantity) AS Orders
    INTO
        [datalake]
    FROM
        [orders] TIMESTAMP BY EventProcessedUtcTime
    GROUP BY ProductID, TumblingWindow(second, 5)
    HAVING COUNT(*) > 1
    ```
3.  Save the query.

#### 3.5. Run the Streaming Job to Aggregate Order Data

1.  Start the `aggregate-orders` Stream Analytics job.
2.  Run the order client again in Cloud Shell:
    ```powershell
    node ~/dp-203/Allfiles/labs/18/orderclient
    ```
3.  In Synapse Studio, on the **Data** page, navigate to your linked Azure Data Lake Storage Gen2 account (`synapsexxxxxxx (primary - datalakexxxxxxx)`) and the `files` container.
4.  After a minute, refresh the view. You should see a folder for the current year, then month, then day.
5.  Select the year folder and create a **New SQL script** to **Select TOP 100 rows**. Set the **File type** to **Text format** and apply.
6.  Modify the query to include the header row:
    ```sql
    SELECT
        TOP 100 *FROM
        OPENROWSET(
            BULK '[https://datalakexxxxxxx.dfs.core.windows.net/files/YYYY/](https://datalakexxxxxxx.dfs.core.windows.net/files/YYYY/)**', -- Replace YYYY with the current year
            FORMAT = 'CSV',
            PARSER_VERSION = '2.0',
            HEADER_ROW = TRUE
        ) AS [result]
    ```
7.  Run the query to view the aggregated order data in five-second windows.
8.  Stop the `aggregate-orders` Stream Analytics job in the Azure portal.

**Important:** After completing this lab, remember to delete the resource group `dp203-xxxxxxx` in the Azure portal to avoid incurring unnecessary costs.
