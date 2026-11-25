# Lakehouse Internals and Execution Demo

Companion notebook for the November 19 guest lecture in JHU EN.601.412/612 Big Data Systems, "From Textbook Clusters to Production Lakehouses."

Everything runs on your laptop with open source tools: Apache Spark, Delta Lake, and DuckDB. No cloud account is needed.

## What's in the notebook

**1.1 What a table format adds**
- A plain Parquet folder vs. a Delta table, and what's inside `_delta_log`
- Per-file min/max statistics and how an engine uses them to skip files
- `OPTIMIZE ... ZORDER BY`: before-and-after data skipping on a selective query
- Copy-on-write deletes vs. deletion vectors (merge-on-read)
- Time travel with `VERSION AS OF`

**1.2 Execution showdown**
- A skewed sort-merge join (40% of rows share one key) and how to spot it in the Spark UI
- Two fixes: AQE skew-join handling and a broadcast join
- The same SQL in DuckDB on the same files and machine, with a sanity check that both engines agree

## Requirements

| Item | Version / amount |
|---|---|
| Java | 17 (Spark 3.5 also accepts 8 or 11) |
| Python | 3.10 or 3.11 (3.12 works with `setuptools`; 3.13+ is not supported by PySpark 3.5) |
| RAM | about 8 GB free at the default data size |
| Disk | about 5 GB free |
| Network | needed on first run only (Delta JARs from Maven, optional DuckDB extension) |

## Setup

1. **Install Java 17** and make sure `JAVA_HOME` points to it.
   - macOS: `brew install openjdk@17`
   - Ubuntu/Debian: `sudo apt install openjdk-17-jdk`
   - Windows: install Eclipse Temurin 17 and select "Set JAVA_HOME" in the installer

   Check: `java -version` should report 17.

2. **Create a virtual environment and install packages.**

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate          # Windows: python -m venv .venv, then .venv\Scripts\activate
   pip install "pyspark==3.5.*" "delta-spark==3.2.*" duckdb pandas jupyterlab setuptools
   ```

   On macOS, use `python3`: there is no `python` command until the environment is active. Check `python3 --version` first. Use Python 3.11 for the smoothest setup. If `python3 --version` shows 3.12 or newer, run `brew install python@3.11` and create the environment with `python3.11 -m venv .venv` instead.

   Keep the pyspark and delta-spark versions paired as shown. To use Spark 4.0 instead, install `pyspark==4.0.*` with `delta-spark==4.0.*`.

3. **Start Jupyter and open the notebook.**

   ```bash
   jupyter lab jhu_lakehouse_demo_oss.ipynb
   ```

4. **Run all cells from the top.** Generating the data takes a few minutes. Data is written to a `jhu_demo_data` folder next to the notebook and rebuilt from scratch each run.

5. **Open the Spark UI** at http://localhost:4040 while the notebook's Spark session is running. The config cell prints the exact address. If port 4040 is taken, Spark uses 4041, 4042, and so on.

## Where to look in the Spark UI

**Data skipping (section 1.1):** go to the **SQL / DataFrame** tab and open the filter query. In the plan graph, expand the **Scan** box at the bottom and compare **number of files read** before and after OPTIMIZE. The notebook's "Data skipping, done by hand" cell prints the same comparison directly from the Delta log.

**Skew (section 1.2, Step 1):** go to the **Stages** tab. The join stage is the completed stage with **200 tasks** and the largest **Shuffle Read**. Click its description and compare the **Max** and **Median** columns in **Summary Metrics** for Duration and Shuffle Read Size / Records. Expand **Event Timeline** to see the single long task. The "Skew, done by hand" cell prints each task's row count.

**After the fixes (Steps 2a and 2b):** in the **SQL / DataFrame** tab, the AQE run shows the join with skewed partitions split, and the broadcast run shows `BroadcastHashJoin` with no shuffle of the events table by key.

## Adjusting the data size

The config cell has three knobs:

| Setting | Default | What it controls |
|---|---|---|
| `N_EVENTS` | 20,000,000 | Rows in the events table. Lower to 5,000,000 on an 8 GB laptop. |
| `N_USERS` | 1,000,000 | Rows in the users table (the join's small side) |
| `HOT_USER_SHARE` | 0.40 | Share of events belonging to the single hot user (the skew) |

The skew and data-skipping effects still show at 5M rows. Absolute timings will differ from the lecture slides depending on your hardware.

## Troubleshooting

**macOS: `command not found: python`**
Use `python3 -m venv .venv`. After `source .venv/bin/activate`, plain `python` and `pip` work. If macOS offers to install "command line developer tools," accept and rerun.

**`ModuleNotFoundError: No module named 'distutils'`** (usually the first time a cell converts Spark results to pandas)
PySpark 3.5 still imports `distutils`, which Python removed in 3.12. Fastest fix: `pip install setuptools` inside the environment, restart the kernel, and rerun from the top. If you're on Python 3.13 or newer, rebuild the environment on 3.11 instead: delete `.venv`, run `brew install python@3.11` (macOS), then `python3.11 -m venv .venv` and reinstall.

**`JAVA_HOME is not set` or `Java gateway process exited before sending its port number`**
Java isn't installed or Spark can't find it. Run `java -version`, set `JAVA_HOME`, and restart the Jupyter kernel.

**`ClassNotFoundException: io.delta.sql.DeltaSparkSessionExtension` or `NoSuchMethodError` mentioning Delta**
The pyspark and delta-spark versions don't match. Reinstall the pair from Setup step 2, then restart the kernel.

**The first cell hangs or fails while downloading**
The Delta JARs come from Maven Central on first run. Check your network or proxy, then rerun. After the first success they're cached in `~/.ivy2`.

**`OutOfMemoryError` or `Python worker exited unexpectedly`**
Lower `N_EVENTS` in the config cell. You can also raise `spark.driver.memory` in the same cell if your machine has room. Restart the kernel after changing either.

**Changing a Spark setting in the config cell has no effect**
The session is created once per kernel. Restart the kernel and rerun from the top.

**`delta extension unavailable here` in the last cell**
That bonus cell downloads a DuckDB extension. It's optional. Everything else in the notebook still works without it.

**The DuckDB result count doesn't match the Delta table**
Make sure you ran the export cell after the DELETE cells. DuckDB reads the exported Parquet, which is a snapshot taken at export time.

**Windows: errors about `winutils.exe` or `HADOOP_HOME`**
Spark on Windows needs Hadoop's `winutils.exe`. The simplest fix is to run the notebook in WSL2 (Ubuntu), which avoids the issue entirely.
