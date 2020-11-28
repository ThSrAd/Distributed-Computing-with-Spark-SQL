# Distributed-Computing-with-Spark-SQL
Author: Thanujhaa Sriee (email: thanujhaa.sriee@gmail.com)</br>

#### Project Description:<br>
This project aims at exploring structured dataset using spark SQL,understanding Spark Internals to increase query performance by caching data, using Spark UI to identify bottlenecks in performance. SQL performance aand  tuning Spark configurations[Executors,Core]. I have used Databricks community edition and a dataset from San Francisco Fire Department for analysis.

<hr>

#### Data Description:<br>
This is a dataset from SanFrancisco Fire department, Calls-For-Service includes all fire units responses to calls.Each record includes the call number, incident number, address, unit identifier, call type, and disposition. All relevant time intervals are included.There are multiple records for each call number.Addresses are associated with a block number, intersection or call box, not a specific address.</br>

The source for this data resides in S3 davis-dsv1071/data. You can access this AWS S3 buckets in Databricks Environment by mounting buckets using DBFS or directly using APIs.

Otherwise download a subset of the Data SF's Fire Department Calls for Service [here] (Enter Link name Here). This dataset is about 85 MB.
The entire dataset can be found on [San Francisco Fire Department Calls for Service](https://data.sfgov.org/Public-Safety/Fire-Department-Calls-for-Service/nuek-vuh3/data)</br>

<hr>

#### Environment:</br>

- Create an account and Login to Databricks Community Edition.

![image](https://user-images.githubusercontent.com/69738890/100491118-ca534c00-30e6-11eb-868b-4bbe8909a818.png)

![image](https://user-images.githubusercontent.com/69738890/100491124-dfc87600-30e6-11eb-9c03-04769a9f226e.png)

![image](https://user-images.githubusercontent.com/69738890/100491155-1e5e3080-30e7-11eb-9410-e0bcd2f7f260.png)

![image](https://user-images.githubusercontent.com/69738890/100491218-75fc9c00-30e7-11eb-9e76-df11afc3892b.png)

<hr>

#### Importing Data Files:</br>

If you are trying to mount the data from the AWS S3 into data bricks edition, Please use the bwlow Scala Code

```
val mountDir = "/mnt/davis"
val source = "davis-dsv1071/data"

if (!dbutils.fs.mounts().map(_.mountPoint).contains(mountDir)) {
  println(s"Mounting $source\n to $mountDir")
  val accessKey = "Enter your access Key Here"
  val secretKey = "Enter your secret Key Here"
  val sourceStr = s"s3a://$accessKey:$secretKey@$source"
  
  dbutils.fs.mount(sourceStr, mountDir)
}
```


unique_valid_searches_{date}.csv contains unique valid searches per day:


![image](https://user-images.githubusercontent.com/69738890/100490743-56637480-30e3-11eb-9143-d42a30d311ab.png)

#### Running Spark SQL Queries

![image](https://user-images.githubusercontent.com/69738890/100490775-b528ee00-30e3-11eb-9ff2-577e85ca8bbe.png)

#### Now look at calls by neighborhood.

![image](https://user-images.githubusercontent.com/69738890/100490854-98d98100-30e4-11eb-99c6-7cf91d46a170.png)

#### Which neighborhoods have the most fire calls?

![image](https://user-images.githubusercontent.com/69738890/100490795-e1dd0580-30e3-11eb-8772-364f48c39ba1.png)

#### Visualizing Data

We use the built-in Databricks visualization to see which neighborhoods have the most fire calls.

![image](https://user-images.githubusercontent.com/69738890/100490813-1c46a280-30e4-11eb-81e0-b8602b62c200.png)

####  Spark Internals - Optimization
 1. CACHING DATA
 Run ```  SELECT count(*) FROM fireCalls```
 Command takes 3.20 seconds to run 
 Now Cache the data
 ![image](https://user-images.githubusercontent.com/69738890/100491620-a98cf580-30ea-11eb-9ac8-4325f4bbffc0.png)

  If we check the Spark UI we will observe that our file in memory takes up ~59 MB, and on disk it takes up ~90 M,

  ![image](https://user-images.githubusercontent.com/69738890/100491866-af83d600-30ec-11eb-9775-72f9b93a4e25.png)

  Run this ```  SELECT count(*) FROM fireCalls``` again
  Conclusion after caching, Command takes just 0.68 seconds to run (Data is deserialized and available in memory in spark,rather than on -disk, this speeds the process)
  
  2. LAZY CACHING 
  Only a chunk of data is avilable in memory
  
  3. SHUFFLING PARTITIONS
  Narrow Transformations: The data required to compute the records in a single partition reside in at most one partition of the parent DataFrame.
  Examples SELECT (columns), DROP (columns), WHERE
  
  Wide Transformations: The data required to compute the records in a single partition may reside in many partitions of the parent DataFrame.
  Examples include:DISTINCT, GROUP BY, ORDER BY
  
  Shuffling results in lot of i/o overload, manually we will be tweaking the spark.sql.shuffle.partitions parameter to controls how many resulting partitions there are after a   shuffle (wide transformation). By default, this value is 200 regardless of how large or small your dataset is, or your cluster configuration.Let's change this parameter to 8
  
  ![image](https://user-images.githubusercontent.com/69738890/100492310-69307600-30f0-11eb-9766-d34789b799da.png)
   
  Run below query:
  ![image](https://user-images.githubusercontent.com/69738890/100492367-ecea6280-30f0-11eb-8812-83de2d9ca991.png)
  
   Time taken when spark.sql.shuffle.partition =8  =>3.75 s
   Time taken when spark.sql.shuffle.partition =64  =>3.28s
   Time taken when spark.sql.shuffle.partition =100  =>3.70s
   Time taken when spark.sql.shuffle.partition =400  =>3.11s
   
   Conclusion: When dealing with small amounts of data, we must reduce the number of shuffle partitions otherwise we will end up with many partitions with small numbers of entries in each partition, which results in underutilization of all executors and increases the time it takes for data to be transferred over the network from the executor to the executor.
On the other hand, when you have too much data and too few partitions, it causes fewer tasks to be processed in executors, but it increases the load on each individual executor and often leads to memory error


How to run this project?
pre-requisites:

Docker and docker-compose must be running on your laptop.
You have credentials for source and destination S3 buckets. (Both are private buckets)
You need to have AWS Redshift cluster endpoint. guide to create Redshift cluster using IaC
Step 1: Once the requirements are met, launch Airflow on your laptop by running: docker-compose up from the location where docker-compose.yml is located.

Shravan: batch-etl$ docker-compose up
Creating network "batch-etl_default" with the default driver
Creating batch-etl_postgres_1 ... done
Creating batch-etl_webserver_1 ... done

webserver_1  |   ____________       _____________
webserver_1  |  ____    |__( )_________  __/__  /________      __
webserver_1  | ____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
webserver_1  | ___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
webserver_1  |  _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
Inside the docker-compose.yml we have the volumes section, which maps our dags directory to airflow's dag-bag: /usr/local/airflow/dags. Next, we map the custom Airflow Plugin that we created to extend Airflow's functionality by adding two custom operators, this is mapped to the airflow's plugin directory. Lastly, inside both my operators, I have made use of s3fs python package, which is essentially a wrapper around boto3 package, but provides more simpler interface. Add s3fs to requirements.txt and map that to /requirements.txt. The reason we need to map this to way is because the entrypoint docker script runs pip install -r requirements.txt from / within the docker container.

volumes:
  - ./street-easy/dags:/usr/local/airflow/dags
  # Uncomment to include custom plugins
  - ./street-easy/plugins:/usr/local/airflow/plugins
  # Additional python packages used inside airflow operators
  - ./street-easy/requirements.txt:/requirements.txt
Step 2:: Configure Airflow Variables Login to Airflow Console: http://localhost:8080/admin , and create two Variables. Our code uses these variables to reference the source and destination buckets. variables

Next, create the following connections:

aws_credentials: (Type: Amazon Web Services, Login:, Password:)
aws_dest_credentials: (Type: Amazon Web Services, Login:, Password:)
redshift: Shown below is the configuration connections
Step 3: There are two dags in our dag-bag: create_postgres_table and street_easy. The first is used to create a table in Redshift. Turn on the create_postgres_table DAG and trigger it manually. Once the dag finishes running, it will create the tables in Redshift. After that, turn on the street_easy dag. This will trigger the execution automatically since the start date is in the past.

Step 4: Launch the jupyter notebook provided here: notebook . Navigate to "Answering Business questions using data" section. Run the code cells.
