ko# PAPU [![](https://img.shields.io/github/license/sourcerer-io/hall-of-fame.svg?colorB=ff0000)](https://github.com/rsanimesh/PAPU/blob/master/LICENSE)  [![](https://img.shields.io/badge/Rahul-Sinha-brightgreen.svg?colorB=ff0000)](http://sinharahul.com/)

Personal Artificial Programmable Unit.


 Since your files are already in a local folder (or DBFS/Volumes) within Databricks, you cannot simply "import" an existing directory into the MLflow UI. MLflow expects runs to be created through the API so it can manage the database entries (metrics, params, and timestamps) and the artifact storage.
To register them, you need to write a small script that iterates through your folders and re-logs them to a new Databricks MLflow experiment.
The Strategy
You will create a loop that:
 * Reads your JSON files (containing your parameters and metrics).
 * Starts a new mlflow.start_run().
 * Logs the data using mlflow.log_param and mlflow.log_metric.
 * Logs the folder itself as an artifact.
Implementation Script
Run this in a Python notebook cell. Replace the paths with your actual folder locations.
import mlflow
import json
import os

# 1. Set your target experiment path in Databricks
experiment_path = "/Users/<your-email>/migrated_runs"
mlflow.set_experiment(experiment_path)

# 2. Path to your downloaded 'mlruns' folder
# If it's on the driver, use local path; if on DBFS, use /dbfs/ prefix
base_dir = "./mlruns/863899491684689248" 

def register_local_runs(source_dir):
    # Iterate through each run folder
    for run_id in os.listdir(source_dir):
        run_path = os.path.join(source_dir, run_id)
        
        # Check if it's a directory (MLflow runs are folders)
        if not os.path.isdir(run_path):
            continue
            
        print(f"Registering run: {run_id}")
        
        with mlflow.start_run(run_name=f"Imported-{run_id}"):
            # --- Log Parameters ---
            params_path = os.path.join(run_path, "params")
            if os.path.exists(params_path):
                for p in os.listdir(params_path):
                    with open(os.path.join(params_path, p), 'r') as f:
                        mlflow.log_param(p, f.read().strip())

            # --- Log Metrics ---
            metrics_path = os.path.join(run_path, "metrics")
            if os.path.exists(metrics_path):
                for m in os.listdir(metrics_path):
                    with open(os.path.join(metrics_path, m), 'r') as f:
                        # Metrics are often logged as 'timestamp value'
                        line = f.read().split()
                        if line:
                            mlflow.log_metric(m, float(line[-1]))

            # --- Log Artifacts (The actual JSON/Model files) ---
            artifacts_path = os.path.join(run_path, "artifacts")
            if os.path.exists(artifacts_path):
                mlflow.log_artifacts(artifacts_path)

# Execute the migration
register_local_runs(base_dir)

Key Points to Watch Out For:
 * Path Formatting: If you used the aws s3 cp command from your previous message, your files are on the Driver node. Use standard local paths (e.g., ./mlruns/...). If you moved them to Unity Catalog or DBFS, use /dbfs/ or /Volumes/.
 * JSON Structure: The script above assumes the standard MLflow mlruns filesystem structure (folders named params, metrics, etc.). If your JSON files are in a custom format, you'll need to use json.load(f) to parse the keys and values before logging.
 * New Run IDs: Note that Databricks will generate new Run IDs for these. I added the original ID to the run_name so you can still search for them.
What if you want to register a Model?
If one of those folders is actually a saved model (contains an MLmodel file), you should use the following after the run is logged:
# Inside the start_run block
mlflow.register_model(f"runs:/{mlflow.active_run().info.run_id}/model_folder_name", "Your_Model_Name")

Would you like me to adjust the script to parse a specific JSON schema if your files aren't in the standard MLflow folder format?

