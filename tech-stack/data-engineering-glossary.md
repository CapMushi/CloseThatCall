# Data Engineering Tech Stack Glossary

A plain-English guide to the technologies used in data engineering. Written for someone seeing these terms for the first time. Technology only; non-technical topics (SDLC, teamwork, stakeholders) live in other files.

**How each entry is laid out:** what it is, why it exists, how it works (only when needed), an optional "Going deeper" for interview-level detail, and one example told step by step.

## Basic ideas used throughout

- **Data pipeline:** a chain of automated steps that moves data from where it is created (an app, a database, a sensor) to where people use it (a report, a dashboard, a machine learning model).
- **Batch vs streaming:** batch means processing data in chunks on a schedule (for example, every night). Streaming means processing each piece of data within seconds of it being created.
- **Data lake:** a cheap, very large storage area that holds raw files of any shape (spreadsheets, logs, JSON). Think of a big warehouse of unsorted boxes.
- **Data warehouse:** a database organized specifically to answer business questions quickly (total sales by region, for example). Think of a well-organized library.
- **ETL / ELT:** Extract (pull data out of a source), Transform (clean and reshape it), Load (put it where it will be used). ELT does the loading first and transforms inside the destination.
- **Schema:** the shape of a table: its column names and the type of data in each (text, number, date).
- **Partition:** splitting a big dataset into smaller pieces (usually by date) so a query only reads the pieces it needs.
- **Cluster:** a group of computers that work together on one big job.

---

## Languages

**Python**
- *What it is:* A general-purpose programming language known for being easy to read and write.
- *Why it exists:* Data pipelines involve many small jobs (call an API, clean a file, trigger another tool). Python is quick to write and has libraries for almost everything, so it is the "glue" of data engineering.
- *Example in a DE project:*
  1. A company's sales system exposes daily sales through a web API.
  2. A Python script calls the API each morning and downloads the data.
  3. It checks that required fields (like order ID) are present and drops broken rows.
  4. It saves the clean file to cloud storage, ready for the next step.

**SQL (Structured Query Language)**
- *What it is:* The standard language for asking questions of data stored in tables: "show me total sales per customer last month."
- *Why it exists:* Most business data lives in tables, and SQL lets you filter, combine, and summarize them without writing a full program.
- *Going deeper:* Interviews often test joins (combining tables), window functions (calculations across related rows, like a running total), and query performance.
- *Example in a DE project:*
  1. There is an `orders` table and a `customers` table.
  2. A SQL query joins them on customer ID.
  3. It uses a window function to add up each customer's spending over time.
  4. The result is saved as a new table that the analytics team reads.

**PySpark**
- *What it is:* A way to write Apache Spark jobs (see Spark below) using Python.
- *Why it exists:* Spark is powerful but was originally written for Java and Scala. PySpark lets Python users run huge jobs across many computers without learning those languages.
- *Example in a DE project:*
  1. Website click data piles up to hundreds of gigabytes, too big for one computer.
  2. A PySpark job splits the data across a cluster of computers.
  3. Each computer removes duplicate clicks from its share.
  4. The results are combined and saved, split by date.

---

## Cloud Platforms

A cloud platform rents you computers, storage, and ready-made services over the internet instead of you buying and running your own hardware. The three big providers offer similar things under different names.

**AWS (Amazon Web Services)**
- *What it is:* Amazon's cloud platform, the largest of the three.
- *Why it exists:* Companies would rather rent storage and computing power, paying only for what they use, than build their own data centers.
- *Example in a DE project:*
  1. Raw files land in S3 (storage).
  2. Glue cleans them (processing).
  3. The clean data is loaded into Redshift (warehouse).
  4. Analysts query Redshift for reports.

**GCP (Google Cloud Platform)**
- *What it is:* Google's cloud platform, especially popular for analytics because of BigQuery.
- *Why it exists:* Same reason as AWS: rent instead of build. Google's strength is large-scale data analysis.
- *Example in a DE project:*
  1. Marketing data is saved to Google Cloud Storage.
  2. A Dataproc job cleans it.
  3. The result is loaded into BigQuery.
  4. A dashboard reads from BigQuery.

**Azure**
- *What it is:* Microsoft's cloud platform, common in companies that already use Microsoft products.
- *Why it exists:* Same reason as AWS and GCP, with tight links to Microsoft tools like Power BI and SQL Server.
- *Example in a DE project:*
  1. Azure Data Factory copies data nightly from an office SQL Server into cloud storage.
  2. Synapse cleans and organizes it.
  3. Power BI reports read the result.

---

## Big Data & Lakehouse

"Big data" means data too large for one computer, so the work is spread across a cluster. A "lakehouse" combines the cheap storage of a data lake with the reliability and speed of a warehouse.

**Apache Spark**
- *What it is:* A tool that processes very large datasets by splitting the work across many computers at once.
- *Why it exists:* One computer would take days to process terabytes of data. Spark divides it into pieces, processes the pieces in parallel, and combines the answers.
- *How it works:* You describe what you want done (filter, join, group). Spark plans the work, hands parts of it to the computers in the cluster, and keeps data in memory where possible to be fast.
- *Going deeper:* A "shuffle" is when data must move between computers (for example during a join or group-by). Shuffles are slow, so much Spark tuning is about avoiding or reducing them. Partitioning strategy matters for the same reason.
- *Example in a DE project:*
  1. A 2 TB table of transactions must be combined with customer and product tables.
  2. Spark spreads the join across the cluster.
  3. It writes the combined result out, split by date.

**Databricks**
- *What it is:* A company platform built around Spark that adds a friendlier workspace: notebooks, job scheduling, and managed clusters.
- *Why it exists:* Setting up and maintaining Spark yourself is hard. Databricks handles it and adds Delta Lake, a storage layer that makes data files behave like reliable tables.
- *Example in a DE project:*
  1. Raw data lands in a "bronze" Delta table exactly as received.
  2. A Databricks job cleans it into a "silver" table.
  3. Another job summarizes it into a "gold" table for reports (see Medallion Architecture).

**Apache Iceberg**
- *What it is:* A "table format": a set of rules and metadata that turns a pile of files in a data lake into something that behaves like a real database table.
- *Why it exists:* Plain files in a lake cannot be safely updated, have no history, and break if the structure changes. Iceberg tracks which files belong to the table so you can change and query it safely.
- *Going deeper:* Iceberg supports ACID transactions (Atomicity, Consistency, Isolation, Durability: in short, changes either fully happen or not at all, and readers never see half-finished work), schema evolution (changing columns without rewriting data), and time travel (querying the table as it was earlier).
- *Don't confuse with:* Hudi and Delta Lake, which solve the same problem with different designs.
- *Example in a DE project:*
  1. A table of customer records sits in S3 as Iceberg.
  2. The team adds a new "loyalty_tier" column.
  3. No existing data is rewritten, and old queries keep working.
  4. If a bad load happens, the team queries yesterday's version to compare.

**Apache Hudi**
- *What it is:* Another table format for data lakes, designed especially for frequent updates and deletes.
- *Why it exists:* Data in a lake is normally add-only. Hudi makes it efficient to change individual rows, which is needed when source databases are constantly updating records.
- *Going deeper:* Hudi offers two storage styles: Copy-on-Write (rewrite files on update; faster reads) and Merge-on-Read (log changes separately and merge later; faster writes).
- *Don't confuse with:* Iceberg, which is stronger at large-scale table management; Hudi is stronger at fast incremental updates.
- *Example in a DE project:*
  1. A customer's address changes in the main database.
  2. The change is captured (see Debezium) and sent to the lake.
  3. Hudi updates just that customer's row instead of rewriting the whole day's data.

**Unity Catalog**
- *What it is:* Databricks' central control panel for who can see what data and where data came from.
- *Why it exists:* With many teams and many tables, you need one place to manage permissions, track lineage (where data came from), and keep an audit record.
- *Example in a DE project:*
  1. The data team owns the "gold" reporting tables.
  2. Unity Catalog is set so everyone can read them but only the data team can change them.
  3. It also records which source tables fed each gold table.

**Amazon EMR (Elastic MapReduce)**
- *What it is:* AWS's service for renting a ready-made cluster that runs Spark and similar tools.
- *Why it exists:* Building your own cluster is slow and costly. EMR creates one on demand and lets you shut it down when the job finishes.
- *Example in a DE project:*
  1. A schema change means a year of history must be reprocessed.
  2. The team starts an EMR cluster and runs a PySpark job overnight.
  3. The cluster is shut down in the morning so it stops costing money.

**GCP Dataproc**
- *What it is:* Google's version of EMR: rentable clusters for Spark and Hadoop.
- *Why it exists:* Same reason as EMR, on Google Cloud.
- *Example in a DE project:*
  1. Raw event logs sit in Google Cloud Storage.
  2. A Spark job on Dataproc cleans and summarizes them.
  3. Daily totals are loaded into BigQuery.

---

## Data Warehousing & Storage

**Amazon S3 (Simple Storage Service)**
- *What it is:* AWS's storage for files of any kind, organized into "buckets" (like top-level folders).
- *Why it exists:* It is cheap, practically unlimited, and reliable, so it is the standard place to keep raw and processed data files, forming the base of most AWS data lakes.
- *Going deeper:* S3 stores "objects" (files) with a key (the path). It has no true folders; a "folder" is just part of the file's name. Data is often laid out by date (`sales/2026/09/25/`) so tools can skip irrelevant files.
- *Example in a DE project:*
  1. A partner sends a daily CSV file.
  2. A script saves it to `s3://company-raw/partner-sales/2026-09-25/`.
  3. Later steps read from that location.

**Snowflake**
- *What it is:* A cloud data warehouse that runs on AWS, Azure, or GCP.
- *Why it exists:* It separates storage from computing power, so many teams can run heavy queries at the same time without slowing each other down, and you can turn computing power up or down as needed.
- *Going deeper:* Compute is provided by "virtual warehouses" (groups of computers you size and pause). Snowflake handles JSON and similar semi-structured data well. Snowpipe loads files automatically as they arrive; Streams and Tasks let you process only new or changed rows on a schedule.
- *Example in a DE project:*
  1. JSON event files arrive in cloud storage.
  2. Snowpipe loads each into a raw table automatically.
  3. A scheduled Task uses a Stream to pick up only the new rows and add them to a summary table.

**Amazon Redshift**
- *What it is:* AWS's data warehouse.
- *Why it exists:* It stores data by column instead of by row, which makes questions like "total sales by month" far faster than in an ordinary database.
- *Going deeper:* Redshift Spectrum lets Redshift query files in S3 directly, without loading them first.
- *Example in a DE project:*
  1. Clean Parquet files sit in S3.
  2. A `COPY` command loads them into Redshift.
  3. Analysts run reports against Redshift tables.

**Google BigQuery**
- *What it is:* Google's data warehouse. You do not manage any servers; you just write SQL.
- *Why it exists:* It removes the work of sizing and running a warehouse. It scales automatically and (in its standard pricing) charges by how much data each query reads.
- *Going deeper:* Because cost depends on data scanned, tables are usually partitioned by date and clustered by common filter columns, so queries read less.
- *Example in a DE project:*
  1. Event data is stored in a table partitioned by date.
  2. A scheduled query summarizes yesterday's events into a daily table.
  3. A dashboard reads that small daily table quickly and cheaply.

**PostgreSQL**
- *What it is:* A free, popular relational database (data stored in tables with rows and columns).
- *Why it exists:* Applications need a reliable place to store their data as it is created and changed, for example customer orders.
- *How it fits in data engineering:* It is usually a source that pipelines copy data from, not the place where analysis happens.
- *Example in a DE project:*
  1. An online shop saves every order into a Postgres `orders` table.
  2. A pipeline copies changes from Postgres to the warehouse each hour.
  3. Analysts study orders in the warehouse without slowing down the shop.

**MongoDB**
- *What it is:* A "NoSQL" database that stores flexible, document-style records (like JSON) instead of fixed tables.
- *Why it exists:* Some data does not fit neatly into rows and columns, such as user profiles where each person has different fields.
- *Example in a DE project:*
  1. User profiles are stored as nested documents in MongoDB.
  2. A job pulls them out on a schedule.
  3. It flattens nested fields into normal columns and loads them into warehouse tables.

**DynamoDB**
- *What it is:* AWS's NoSQL database that stores items by a key and returns them extremely quickly, at any scale.
- *Why it exists:* Applications like shopping carts need instant lookups for millions of users at once.
- *Going deeper:* Each item is found by its partition key. DynamoDB Streams records every change so other systems can react to it.
- *Example in a DE project:*
  1. Orders are written to DynamoDB.
  2. DynamoDB Streams emits each change.
  3. A Lambda function forwards the changes to a streaming service for analytics.

**Parquet**
- *What it is:* A file format that stores data column by column and compresses it.
- *Why it exists:* Analysis usually needs only a few columns. Storing by column lets tools read just those and skip the rest, making queries faster and files smaller than CSV.
- *Example in a DE project:*
  1. A Spark job writes results as Parquet files split by date.
  2. A query for "last week only" reads just the matching date folders.
  3. It reads only the columns it needs, saving time and cost.

**Azure Synapse Analytics**
- *What it is:* Microsoft's analytics service combining a data warehouse with Spark processing in one workspace.
- *Why it exists:* It lets teams on Azure clean big data and serve it for reporting without stitching several products together.
- *Example in a DE project:*
  1. Data Factory lands raw files in Azure storage.
  2. A Synapse Spark notebook cleans them.
  3. The clean tables are stored in Synapse and read by Power BI.

---

## Orchestration & ETL

Orchestration means deciding what runs when, in what order, and what to do when something fails.

**Apache Airflow**
- *What it is:* A tool for scheduling and monitoring data workflows that you define in Python.
- *Why it exists:* Real pipelines have many steps that depend on each other (step 3 can only start after steps 1 and 2 succeed). Airflow runs them in order, retries failures, and shows what happened.
- *How it works:* A workflow is called a DAG (Directed Acyclic Graph): a set of tasks with arrows showing what must finish before what, and no loops. The scheduler starts tasks when their turn comes.
- *Going deeper:* Sensors wait for something to happen (like a file arriving). Retries and alerts handle failures. Backfilling reruns a DAG over past dates.
- *Example in a DE project:*
  1. At 2 a.m. Airflow waits for the daily file to arrive in S3.
  2. Once it arrives, it starts a Glue job to clean it.
  3. When cleaning succeeds, it runs dbt to build reporting tables.
  4. If any step fails, it retries and then posts an alert in Slack.

**dbt (data build tool)**
- *What it is:* A tool that lets you build and manage data transformations using only SQL, inside your warehouse.
- *Why it exists:* Transformations used to be scattered scripts with no testing or documentation. dbt treats them like software: versioned, tested, and documented.
- *How it works:* Each transformation is a SQL `SELECT` saved in a file (a "model"). dbt runs them in the right order and creates the tables or views.
- *Going deeper:* Models are usually layered: staging (light cleanup), intermediate (joins), marts (final tables for reports). Built-in tests check things like "this column has no nulls" and "these values are unique."
- *Example in a DE project:*
  1. A staging model cleans the raw orders table.
  2. A mart model joins orders to customers and totals revenue.
  3. `dbt test` confirms order IDs are unique and never empty before analysts use the table.

**AWS Glue**
- *What it is:* AWS's managed service for cleaning and moving data, with no servers for you to run.
- *Why it exists:* It lets teams run data-processing jobs (built on Spark) and keep a catalog of what data exists, without managing infrastructure.
- *Going deeper:* A Crawler scans files and records their structure in the Glue Data Catalog, so other AWS tools (like Athena) can query them by name.
- *Example in a DE project:*
  1. Nightly CSV files arrive in S3.
  2. A Glue job converts them to Parquet, split by date.
  3. A Crawler registers the new table so analysts can query it immediately.

**Azure Data Factory (ADF)**
- *What it is:* Azure's service for building pipelines that copy and transform data between many systems, mostly through a visual editor.
- *Why it exists:* Companies need a managed way to move data from office systems and other clouds into Azure on a schedule.
- *Example in a DE project:*
  1. Every night ADF copies new rows from an office SQL Server into Azure storage.
  2. When the copy finishes, it triggers a Synapse job.
  3. The job builds the reporting tables.

---

## Streaming & CDC

**CDC (Change Data Capture)** means detecting each change made to a database (new, updated, or deleted rows) and sending it onward, instead of re-copying the whole database.

**Apache Kafka**
- *What it is:* A system for passing streams of events (small messages like "order placed") between applications in real time.
- *Why it exists:* Without it, every system would have to connect directly to every other one. With Kafka, producers publish events once and any number of consumers read them independently.
- *How it works:* Events are written to named "topics." Producers write to a topic; consumers read from it at their own pace. Events are kept for a set time, so they can be re-read.
- *Going deeper:* Topics are split into partitions for scale and ordering (order is guaranteed within a partition). Consumer groups let several consumers share the work of one topic.
- *Don't confuse with:* Kinesis, AWS's managed alternative to Kafka.
- *Example in a DE project:*
  1. The order service publishes an event for each new order to an "orders" topic.
  2. A fraud-detection job reads the topic and scores each order.
  3. A separate job reads the same topic and stores events in the data lake.

**Debezium**
- *What it is:* A tool that watches a database and turns every change into an event.
- *Why it exists:* Repeatedly asking a database "what changed?" is slow and puts load on it. Debezium reads the database's own change log, so it catches everything without slowing the database.
- *Example in a DE project:*
  1. A customer updates their address in a Postgres database.
  2. Debezium reads the change from the database log and sends it to Kafka.
  3. Downstream systems update their copies within seconds.

**Apache Flink**
- *What it is:* A tool for processing streams of events continuously, with very low delay.
- *Why it exists:* Some questions cannot wait for a nightly batch, such as "is this card being used fraudulently right now?"
- *Going deeper:* Flink keeps "state" (memory of past events) and works with windows of time (like "the last 5 minutes"). It can process by when an event actually happened, not just when it arrived.
- *Example in a DE project:*
  1. Card transactions flow through Kafka.
  2. Flink counts each card's transactions over a rolling 5-minute window.
  3. A card with unusually many is flagged as suspicious immediately.

**Amazon Kinesis**
- *What it is:* AWS's managed service for streaming data in real time.
- *Why it exists:* It does the same job as Kafka without you having to run the servers.
- *Going deeper:* Data streams are divided into shards, each with a fixed capacity. Kinesis Data Firehose delivers stream data to places like S3 automatically.
- *Example in a DE project:*
  1. Sensors send readings to a Kinesis stream.
  2. A Lambda function checks each reading for problems.
  3. Firehose saves all readings to S3 for later analysis.

---

## Serverless & ML Compute

**AWS Lambda**
- *What it is:* A service that runs a small piece of code only when something triggers it, with no server for you to manage.
- *Why it exists:* Many small jobs only need to run occasionally. With Lambda you pay only while the code runs.
- *Going deeper:* Each run has a time limit (up to 15 minutes), so Lambda suits short tasks, not big processing.
- *Example in a DE project:*
  1. A file is uploaded to S3.
  2. That upload triggers a Lambda function.
  3. It checks the file's format and moves it to the right folder, or sends an alert if it is wrong.

**AWS SageMaker**
- *What it is:* AWS's service for building, training, and running machine learning models.
- *Why it exists:* Training models needs powerful computers and lots of setup. SageMaker provides them ready to use.
- *How it fits in data engineering:* Data engineers build the pipelines that deliver clean, reliable data (called "features") that models learn from.
- *Example in a DE project:*
  1. A pipeline gathers customer activity data into S3.
  2. A SageMaker job trains a model to predict which customers will leave.
  3. The predictions are saved back for the business to use.

---

## Data Quality & Governance

**Great Expectations**
- *What it is:* A tool for writing automatic checks on data, such as "this column should never be empty."
- *Why it exists:* Bad data quietly breaks reports. Automatic checks catch problems before people rely on the numbers.
- *Example in a DE project:*
  1. After a new file is loaded, a set of checks runs.
  2. The checks confirm order IDs are present and amounts are positive.
  3. If a check fails, the pipeline stops and alerts the team.

**Schema Enforcement**
- *What it is:* Refusing data that does not match the expected structure.
- *Why it exists:* If a source silently changes a column's type or adds unexpected fields, it can corrupt tables downstream. Rejecting bad data early keeps tables trustworthy.
- *Example in a DE project:*
  1. A table expects `amount` to be a number.
  2. A source starts sending it as text.
  3. The write is rejected and the team is alerted, instead of the table filling with bad values.

**RBAC (Role-Based Access Control)**
- *What it is:* Giving people permissions through roles (such as "analyst" or "engineer") instead of one person at a time.
- *Why it exists:* It keeps access organized and safe: people can see only what their job needs, and permissions are easy to change in one place.
- *Example in a DE project:*
  1. An "analyst" role is allowed to read reporting tables only.
  2. An "engineer" role can also write to raw and staging tables.
  3. A new analyst is added to the role and gets the right access automatically.

**Data Lineage**
- *What it is:* A record of where data came from and every step it went through to reach its current form.
- *Why it exists:* When a number looks wrong, lineage lets you trace it back to the source. It also shows what would break if a source changes.
- *Example in a DE project:*
  1. A dashboard number looks wrong.
  2. The lineage view shows it comes from a summary table, built from a cleaned table, built from a raw source.
  3. The team finds the fault in the source in minutes.

**SLA (Service Level Agreement)**
- *What it is:* A promise about how reliable or timely something will be, such as "the sales report is ready by 6 a.m. every day."
- *Why it exists:* Other people plan their work around your data. An SLA sets clear expectations and tells you when you are failing them.
- *Example in a DE project:*
  1. The SLA says the daily load must finish by 6 a.m.
  2. The pipeline is set to alert if it has not finished by then.
  3. The on-call engineer investigates before business users notice.

**Compliance**
- *What it is:* Following the laws and rules about how data must be stored, protected, and used.
- *Why it exists:* Mishandling sensitive data can cause legal penalties and harm people. Common rules include GDPR (protects personal data of people in Europe), HIPAA (protects US health information), SOX (financial reporting integrity), and PCI-DSS (protects payment card data).
- *Example in a DE project:*
  1. Incoming data contains card numbers.
  2. The pipeline replaces each number with a random token before storing it.
  3. Reports and analysts never see the real numbers.

---

## DevOps & Infrastructure

DevOps is the set of tools and habits for building, testing, and running software reliably. In data engineering it covers how pipelines and their servers are set up and released.

**Terraform**
- *What it is:* A tool that creates cloud resources (storage, servers, permissions) from configuration files you write.
- *Why it exists:* Clicking through a cloud console is slow and error-prone, and cannot be repeated exactly. Files can be reviewed, saved, and reused.
- *How it works:* You describe the setup you want. Terraform compares it with what exists and makes only the needed changes.
- *Going deeper:* This approach is called Infrastructure as Code. Terraform keeps a "state" file recording what it created. `plan` previews changes and `apply` performs them.
- *Example in a DE project:*
  1. A file describes a storage bucket, a permission role, and a Glue job.
  2. Running Terraform builds them in the test environment.
  3. The same file builds an identical copy in production.

**Docker**
- *What it is:* A tool that packages an application and everything it needs into a single unit called a container.
- *Why it exists:* "It works on my computer" is a classic problem. A container runs the same way everywhere.
- *Going deeper:* An image is the packaged blueprint; a container is a running copy of it.
- *Example in a DE project:*
  1. A PySpark job needs specific library versions.
  2. They are packed into a Docker image.
  3. The same image runs on a laptop, in testing, and in production.

**Kubernetes**
- *What it is:* A system that runs, scales, and repairs many containers across a group of computers.
- *Why it exists:* Running one container is easy; running hundreds reliably needs automation to restart failures and add capacity.
- *Going deeper:* Containers run inside "pods," the smallest unit Kubernetes manages.
- *Example in a DE project:*
  1. Airflow is set to start each task in its own pod.
  2. When many tasks queue up, Kubernetes starts more pods.
  3. When they finish, the pods disappear and the resources are freed.

**GitHub Actions**
- *What it is:* Automation built into GitHub that runs steps whenever something happens, such as new code being submitted.
- *Why it exists:* It automatically checks changes so mistakes are caught before they reach production.
- *Example in a DE project:*
  1. An engineer proposes a change to a dbt model.
  2. GitHub Actions automatically runs tests and style checks.
  3. The change can only be merged if all checks pass.

**Jenkins**
- *What it is:* A self-hosted automation server that builds, tests, and deploys code.
- *Why it exists:* Same purpose as GitHub Actions; it is older and very customizable, and many companies still run it on their own servers.
- *Example in a DE project:*
  1. Code is merged to the main branch.
  2. Jenkins builds a Docker image and runs unit tests.
  3. If the tests pass, it deploys the new version of the pipeline.

**CI/CD (Continuous Integration / Continuous Delivery or Deployment)**
- *What it is:* An approach where every code change is automatically tested (CI) and then automatically released (CD).
- *Why it exists:* Manual testing and releasing is slow and error-prone. Automation makes releases small, frequent, and safe.
- *Example in a DE project:*
  1. An engineer changes a transformation.
  2. Tests run automatically (CI).
  3. On success, the change is released to the test environment, then to production (CD).

**GitOps**
- *What it is:* Managing infrastructure and deployments through Git, so the Git repository is the single source of truth.
- *Why it exists:* Everything is recorded, reviewed, and reversible, because the only way to change production is to change the repository.
- *Example in a DE project:*
  1. An engineer edits a Terraform file and opens a pull request.
  2. A teammate reviews it.
  3. After it is merged, automation applies the change; nobody edits production by hand.

---

## Observability & Monitoring

Observability means being able to see what your systems are doing and why. Monitoring is watching those signals and alerting when something goes wrong.

**Prometheus**
- *What it is:* A free tool that collects numbers about how systems are performing (metrics) and stores them over time.
- *Why it exists:* You cannot fix what you cannot measure. Prometheus regularly asks each system for its numbers, such as how long a job took.
- *Example in a DE project:*
  1. Airflow publishes numbers like task duration and failures.
  2. Prometheus collects them every minute.
  3. An alert fires if failures spike.

**Grafana**
- *What it is:* A tool for turning metrics into charts and dashboards.
- *Why it exists:* Raw numbers are hard to read; a chart shows trends and problems at a glance.
- *Example in a DE project:*
  1. Grafana reads pipeline metrics from Prometheus.
  2. A dashboard shows job durations, failures, and rows loaded per day.
  3. The team checks it each morning to spot problems.

**Datadog**
- *What it is:* A paid, all-in-one monitoring service for metrics, logs, and traces (the path of a request through a system).
- *Why it exists:* It brings everything into one product so teams do not build their own monitoring stack.
- *Example in a DE project:*
  1. Datadog tracks the number of rows loaded each day.
  2. If today's count is far below the recent average, it pages the on-call engineer.
  3. The engineer finds a broken source before reports go out wrong.

**OpenTelemetry**
- *What it is:* An open standard for collecting logs, metrics, and traces from applications in the same way, whatever tool displays them.
- *Why it exists:* Without a standard, switching monitoring tools means rewriting how every application reports data.
- *Example in a DE project:*
  1. A custom ETL service is instrumented once with OpenTelemetry.
  2. Its data can be sent to Datadog today.
  3. If the company changes tools later, no application code needs to change.

---

## Data Modeling & Architecture

**Dimensional Modeling**
- *What it is:* A way of organizing warehouse tables into "facts" (events you measure, like sales) and "dimensions" (the context, like customer, product, date).
- *Why it exists:* It makes data easy for business users to understand and fast to query. The common layout is a "star schema": one fact table in the middle with dimension tables around it.
- *Example in a DE project:*
  1. A `fct_orders` table records each order and its amount.
  2. It links to `dim_customer`, `dim_product`, and `dim_date` tables.
  3. An analyst can now total revenue by product, region, or month with simple queries.

**Medallion Architecture**
- *What it is:* A way of organizing a lakehouse into three layers: bronze (raw), silver (cleaned), and gold (ready for reports).
- *Why it exists:* Keeping raw data untouched lets you replay and fix mistakes, while each layer improves quality step by step.
- *Example in a DE project:*
  1. Bronze: raw order data is stored exactly as received.
  2. Silver: duplicates are removed and data types corrected.
  3. Gold: totals per day and region are prepared for dashboards.

---

## BI & Analytics

BI (Business Intelligence) tools let people explore data and build charts and dashboards without writing code.

**Looker**
- *What it is:* A BI tool where the business's metrics are defined once in a modeling language called LookML.
- *Why it exists:* Different teams often calculate the same number (like "revenue") differently. Defining it once keeps every dashboard consistent.
- *Example in a DE project:*
  1. The team defines "revenue" once in LookML.
  2. Every dashboard uses that definition.
  3. Sales and finance now report the same number.

**Tableau**
- *What it is:* A BI tool known for drag-and-drop, highly visual dashboards.
- *Why it exists:* It lets non-programmers explore data and build rich charts themselves.
- *Example in a DE project:*
  1. Tableau connects to a Snowflake table of sales.
  2. A manager builds a regional sales dashboard by dragging fields onto a chart.
  3. Stakeholders filter by region without writing any code.

**Power BI**
- *What it is:* Microsoft's BI tool, tightly connected to Azure and Microsoft Office.
- *Why it exists:* Companies on Microsoft products get dashboards that fit into tools they already use.
- *Example in a DE project:*
  1. Power BI connects to tables in Azure Synapse.
  2. It refreshes each morning after the pipeline finishes.
  3. Executives see up-to-date KPIs (key performance indicators, the main numbers a business tracks) on one page.
