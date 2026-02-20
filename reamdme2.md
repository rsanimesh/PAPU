This is a great question. When dealing with billions of rows, running standard profiling queries (like exact distinct counts) can easily crash your driver or lead to queries that spin forever.
Since you are in Databricks, the engine (Apache Spark) is perfectly designed for this, provided you use the right functions.
Here is the best, most optimized way to extract the schema, null counts, total counts, and unique values for your nested Parquet files without breaking your system.
The Strategy for Billions of Rows
 * Read the Schema from Metadata: Parquet files store schema (column names and data types) in their metadata. Spark reads this almost instantly without scanning the actual data.
 * Use Distributed Aggregation: Do not use Pandas or .collect() on the raw data. Let Spark's distributed engine calculate the metrics across your worker nodes.
 * Use approx_count_distinct: This is the most crucial step. Doing an exact COUNT(DISTINCT column) on billions of rows requires massive data shuffling and is the #1 cause of Out-Of-Memory (OOM) errors. Spark's approx_count_distinct uses the HyperLogLog algorithm to give you an answer with ~5% error margin but runs exponentially faster and uses a fraction of the memory.
The Optimized PySpark Solution
You can run this code in a Databricks Notebook attached to a cluster with access to that Unity Catalog volume. Based on your screenshot, I have pre-filled the Volume path.
from pyspark.sql.functions import col, count, when, approx_count_distinct
from pyspark.sql.types import StructType

# Base path from your screenshot
base_volume_path = "/Volumes/catalog_buk_di_prd_an/volume_schema_buk_di_prd_an/external_volume_buk_feature_store/"

# 1. Get the list of folders in the volume
folders = dbutils.fs.ls(base_volume_path)

for folder in folders:
    if folder.isDir():
        folder_name = folder.name.rstrip('/')
        folder_path = folder.path
        
        print(f"==================================================")
        print(f"Profiling Folder: {folder_name}")
        print(f"==================================================")

        try:
            # 2. Read the Parquet data (this is lazy, it doesn't load data into memory yet)
            df = spark.read.parquet(folder_path)

            # 3. Get total count
            total_count = df.count()
            print(f"Total Rows: {total_count}\n")

            if total_count > 0:
                # 4. Build the aggregation expressions dynamically for all columns
                exprs = []
                for c_name, c_type in df.dtypes:
                    # Count Nulls
                    exprs.append(count(when(col(c_name).isNull(), c_name)).alias(f"{c_name}_nulls"))
                    # Approximate Unique Values (Saves your cluster from crashing)
                    exprs.append(approx_count_distinct(col(c_name)).alias(f"{c_name}_uniques"))

                # 5. Execute the aggregation (This is where the actual cluster work happens)
                # We collect only the single row of aggregated results back to the driver
                stats_row = df.agg(*exprs).collect()[0]

                # 6. Print the results in a clean table format
                print(f"{'Column Name':<35} | {'Data Type':<15} | {'Null Count':<15} | {'Approx Uniques':<15}")
                print("-" * 85)
                
                for c_name, c_type in df.dtypes:
                    null_count = stats_row[f"{c_name}_nulls"]
                    unique_count = stats_row[f"{c_name}_uniques"]
                    print(f"{c_name:<35} | {c_type:<15} | {null_count:<15} | {unique_count:<15}")
                    
            else:
                print("Folder is empty or contains no Parquet data.")
            print("\n")

        except Exception as e:
            print(f"Skipping {folder_name} - Could not read as Parquet. Error: {e}\n")

Why this won't break your system:
 * Lazy Evaluation: spark.read.parquet() only looks at the file footers to get the column names and types. It does not load billions of rows into memory.
 * Single Pass Scan: The .agg(*exprs) line bundles all the null checks and unique counting into a single scan of the data.
 * Minimal Driver Overhead: The collect()[0] at the very end only returns one single row of aggregated numbers to your notebook screen, meaning your driver node won't run out of memory.
An Alternative: Databricks Data Explorer (UI)
If you don't want to run code every time, you could register these Parquet folders as External Tables in Unity Catalog. Once they are registered as tables, you can click on them in the Databricks Catalog UI, go to the "Data" tab, and click "Data Profile". Databricks will automatically run a safe, optimized profiling job in the background and show you visual charts of nulls, uniques, and distributions.
Would you like me to show you the SQL commands to register these volume folders as external tables so you can use the built-in UI profiler?
