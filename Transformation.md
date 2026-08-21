# Transformation Layer

Now that we have successfully extracted the data and stored it in our SQL database, it’s time to move on to **Azure Databricks** for data transformation.


| What is Azure Databricks? |
|-------------|
| Azure Databricks is a cloud-based big data and AI platform that enables scalable data processing, machine learning, and collaborative analytics using Apache Spark. |
  

## Setting Up Azure Databricks

To begin, I navigated to **Azure Databricks** and clicked on **Create**. Here are the details I provided:

- **Workspace Name**: shayanathif-databricks-workspace
- **Pricing Tier**: Trial Premium (Standard would have worked too, but Premium gives more features)
- **Managed Resource Group**: managed-azure-project

  
I then clicked on **Review + Create** and deployed the workspace successfully.

After deployment, I clicked on **Go to Resource**, which opened the **Databricks workspace** in a new tab.

  
## Creating a Unity Metastore

| What is a Unity Metastore? |
|-------------|
| It is a storage management system that allows Databricks to use Unity Catalog, which helps manage access to data across different workspaces. |

| What is Unity Catalog? |
|-------------|
| Simply put, **Unity Catalog is a governance tool** that provides centralized access control for data in Databricks. It ensures that the right users have access to the right data securely.



### Setting Up the Metastore

To enable **Unity Catalog**, I created a **Unity Metastore** and attached it to my Databricks workspace. As part of this, I also had to create an **Access Connector**.

### What is an Access Connector?

Think of it as a bridge that lets **Databricks access Azure Data Lake**. Without this, Databricks and Data Lake cannot communicate.

  
After setting this up, I went to **IAM (Identity and Access Management)** in **Azure Data Lake** and added a role to this access connector.

I also created a **new container named ****`unitymetastore`** specifically for the **metastore** and successfully linked it to my workspace. At this point, **Unity Catalog was enabled**.

## Setting Up External Locations

Next, I created three **external locations** to manage different phases of the data pipeline:

- **Bronze** (Raw data storage)
- **Silver** (Transformed data)
- **Gold** (Aggregated and final processed data)

  
These locations require a **storage credential**, which in this case is **the Access Connector** created earlier.

I then went to **Catalog > External Data**, filled in the necessary details, and created the external data sources. Finally, I linked the **external locations** to the corresponding containers in **Azure Data Lake**.

## Creating a Cluster

The last setup step was to create a **Databricks Cluster** in the **Compute** tab. One key thing to check here was ensuring that **Unity Catalog was enabled** in the **Summary** section.


## Data Transformation in Databricks

Now that the setup was complete, I proceeded to **data transformation**.

### Creating a Notebook

I went to **Workspace** and created a new **notebook named ****`silver-notebook`**, where I would perform all transformations.

### Reading Data from Bronze Container

To start, I loaded the data from the **Bronze** container using the following SQL code:

```python
from pyspark.sql.functions import col, split, sum as spark_sum

df = spark.read.format('parquet')\
            .option('inferschema', True)\
            .load('abfss://bronze://shayanathifdatalake.dfs.core.windows.net/rawdata')
```

  ![image](https://github.com/user-attachments/assets/edb6b3b2-bfeb-4b0e-9d3b-45453ef7a4d5)
  
### Transformations

#### 1. Creating a New Column `Model_category` from `Model_ID`

I needed to extract the **first part of ****`Model_ID`** to create a **`Model_category`** column.

```python
df = df.withColumn('Model_category', split(col('Model_ID'), '-')[0])
```

#### 2. Creating a `RevenuePerUnit` Column

This column was created by dividing **Revenue** by **Units Sold**.

```python
df = df.withColumn('RevenuePerUnit', col('Revenue') / col('Units_Sold'))
```


### Answering Business Questions

Now, I performed some queries to gain insights from the data.

#### 1. Which year had the highest revenue?

```python
df.groupBy("Year") \
  .agg(spark_sum("Revenue").alias("Total_Revenue")) \
  .orderBy(col("Total_Revenue").desc()) \
  .limit(1) \
  .show()
```

  ![image](https://github.com/user-attachments/assets/e2f842ac-818c-4f76-9bba-dd193e9c144d)
  
This code groups the data by **Year**, sums up the **Revenue**, sorts it in descending order, and displays the top year.

#### 2. What is the average revenue per unit for each model category?

```python
df.groupBy("Model_category") \
  .agg((spark_sum("Revenue") / spark_sum("Units_Sold")).alias("Avg_Revenue_Per_Unit")) \
  .orderBy(col("Avg_Revenue_Per_Unit").desc()) \
  .display()
```

This groups data by **Model Category**, calculates **average revenue per unit**, and sorts it.

#### 3. Total Units Sold by Every Branch in Every Year (Ascending Order)

```python
df.groupBy('Year', 'BranchName')\
  .agg(spark_sum('Units_Sold').alias('Total_Units'))\
  .sort('Year', 'Total_Units', ascending=[1, 0])
```


  
This groups data by **Year and BranchName**, sums up **Units Sold**, and sorts by **Year (ascending)** and **Total Units (descending)**.

## Visualizations

After these transformations, I created some **visualizations** (e.g., revenue trends, units sold per branch, etc.) to analyze data patterns.


  
## Storing Transformed Data in Silver Layer

Finally, I saved the transformed data in the **Silver** container using the following code:

```python
df.write.format('parquet')\
    .mode('append')\
    .option('path', 'abfss://silver@shayanathifdatalake.dfs.core.windows.net/carsales')\
    .save()
```
| What is Parquet file format? |
|-------------|
| A columnar storage file format optimized for big data processing and analytics. It supports efficient compression and encoding schemes. |

## Transformation Phase Completed ✅

At this point, the **Silver layer** (transformation layer) of the pipeline was successfully completed! 🎉

Next, we will move on to **data aggregation and final storage in the Gold layer**.

[➡ Go to the Gold Layer](Load.md)

