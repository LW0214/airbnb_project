# Readme.md

Note: warnings occured while running are normal, they are the reminder that the files are too large to process.

## Data Ingest

### Download Data

1. Download San Francisco Airbnb Review & Pricing Data: 
    1. Go to http://insideairbnb.com/get-the-data/
    2. Find “San Francisco, California, United States”
    3. Download the first file in the list, “listings.csv.gz”, and unzip the file locally to get `listings.csv`
2. Download police_short.csv
    1. go to https://data.sfgov.org/Public-Safety/Police-Department-Incident-Reports-Historical-2003/tmnf-yvry
    2. select ‘export’, ‘csv’, download and rename as `police_short.csv`
3. Download business.csv
    1. Go to https://data.sfgov.org/Economy-and-Community/Registered-Business-Locations-San-Francisco/g8m3-pdis
    2. click “export”, click “csv”
    3. rename the file to `business.csv`
    

### Upload Data (We have already uploaded them for you and shared them with you in the folder)

- Please change it according to your home directory:
- scp listings.csv zg978@peel.hpc.nyu.edu:/home/zg978/
- scp business.csv zg978@peel.hpc.nyu.edu:/home/zg978/
- scp police_short.csv zg978@peel.hpc.nyu.edu:/home/zg978/
- hdfs dfs -put police_short.csv team3
- hdfs dfs -put business.csv team3
- hdfs dfs -put listings.csv team3

## ETL - Clean data with PySpark

### listings.csv:

- upload `clean_airbnb.py` file to peel:
- run (please change the directory according to your computer)
    
    scp etl_code/zg978PaytonGuo/clean_airbnb.py zg978@peel.hpc.nyu.edu:/home/zg978
    
- run `module load python/gcc/3.7.9`
- run `PYTHONSTARTUP=clean_airbnb.py pyspark --deploy-mode client`
- quit the shell with `quit()` and run:
    
    hdfs dfs -mv team3/airbnb.csv team3/airbnb_old.csv
    
- get in pyspark:
    
    module load python/gcc/3.7.9
    pyspark --deploy-mode client
    
- run the following code in pyspark shell:
    
    import pandas as pd
    airbnb = spark.read.options(header ='True',inferSchema='True', delimiter=',').option("multiLine",'True').option("escape","\"").csv("team3/airbnb_old.csv")
    airbnb = airbnb.toPandas()
    airbnb['price'] = airbnb['price'].astype("string").apply(lambda x: x+',')
    airbnb=spark.createDataFrame(airbnb)
    airbnb.coalesce(1).write.format('com.databricks.spark.csv').options(header='true').save("team3/airbnb.csv")
    
- exit shell with `quit()` and run the following command in hdfs:
    
    hdfs dfs -mkdir team3/etl_code
    
    hdfs dfs -cp team3/airbnb.csv team3/etl_code
    
- The output has been stored as `team3/airbnb.csv` on hdfs

### police_short.csv:   please adjust the file paths as you see fit on your computer

Step 1-4 is unnecessary if you downloaded the first 6000 rows. But if you downloaded the entire dataset, you can follow the instructions below to truncate it. 

- Truncate the csv since it’s too large to convert to pandas dataframe in pyspark by running the below code:
    
    module load python/gcc/3.7.9
    pyspark --deploy-mode client
    
    input_path="team3/police_short.csv"
    data=spark.read.options(header='True',inferSchema='True',delimiter=',').csv(input_path)
    data=data.limit(6000)
    
- drop useless columns in `airbnb_police`.csv
    
    import pandas as pd
    
    data=data.toPandas()
    
    data = data.iloc[:,:14]
    data=data.drop(["Time","Resolution"], axis = 1)
    
    data=spark.createDataFrame(data)
    
    data.coalesce(1).write.format('com.databricks.spark.csv').options(header='true').save('team3/etl_code/police_short_cleaned.csv')
    
    data.coalesce(1).write.format('com.databricks.spark.csv').options(header='true').save('team3/police_short_cleaned.csv')
    
- quit the shell with `quit()` and delete the original police_short.csv file
    
    hdfs dfs -rm -r team3/police_short.csv
    
- rename  `team3/police_short_cleaned.csv` to `team3/police_short.csv`
    
    hdfs dfs -mv team3/police_short_cleaned.csv team3/police_short.csv
    
    hdfs dfs -mv team3/etl_code/police_short_cleaned.csv team3/etl_code/police_short.csv
    
- Upload the cleaning2.py to clean the police_short.csv
    - put police_short in the right location
        
        hdfs dfs -mkdir team3/data_ingest
        hdfs dfs -cp team3/police_short.csv team3/data_ingest/police_short.csv
        
    - upload the py file to clean the police_short.csv; Note the file path is different on your computer
        
        scp etl_code/mc7787MingCao/cleaning2.py  mc7787@peel.hpc.nyu.edu:/home/mc7787
        
- Run cleaning2.py in home on peel, move the output data.csv into team3 folder on peel:
    
    module load python/gcc/3.7.9
    
    PYTHONSTARTUP=cleaning2.py pyspark --deploy-mode client
    
- quit with `quit()`  and upload data.csv onto hdfs and adjust locations
    
    hdfs dfs -put data.csv team3/etl_code/data.csv
    

### business.csv cleaning :

- upload to peel (change according to your directory): `scp etl_code/wm1077LucyWu/business_cleaning.py`
- run:
    
    module load python/gcc/3.7.9
    
    PYTHONSTARTUP=business_cleaning.py pyspark --deploy-mode client
    
- Resulting directories: team3/data_ingest/business_cleaned.csv

## ETL2- Join three datasets, Clean null values

### Join police_short.csv and airbnb.csv

- run `rm -r team3/etl_code/airbnb_police.csv`
- Upload and run cleaning3.py
    - Upload the cleaning3.py; Note the filepath on your computer is different
    `scp etl_code/mc7787MingCao/cleaning3.py mc7787@peel.hpc.nyu.edu:/home/mc7787`
    - run the below lines on home to provide a location for the output file on peel
        
        mkdir team3
        
        mkdir team3/etl_code
        
    - run the below lines:
        
        module load python/gcc/3.7.9
        
        PYTHONSTARTUP=cleaning3.py pyspark --deploy-mode client
        
    - this should result in a `airbnb_police.csv` at location `team3/etl_code/airbnb_police.csv` on peel home, please check if this file exists in that location with `ls team3/etl_code/.` That should result in a `team3/etl_code/airbnb_police.csv` on your home
    - this should also output several png files on peel
- quit with `quit()` and upload the resulting airbnb_police.csv to onto two hdfs locations to be safe:
    
    hdfs dfs -put team3/etl_code/airbnb_police.csv team3/etl_code
    
    hdfs dfs -put team3/etl_code/airbnb_police.csv team3 
    

### Join  `airbnb_police.csv` and `business_cleaned.csv` together: as final_join.csv

- scp `etl_code/wm1077LucyWu/final_join.py` to peel (directory is specific to your computer)
- run:
    
    module load python/gcc/3.7.9
    PYTHONSTARTUP=final_join.py pyspark --deploy-mode client
    
- Output directory: `team3/final_join.csv` on hdfs
- Note: `team3/final_join.csv` is the final joined dataset that we will perform analysis on; it is devoid of any nan values

### **Modify a Column Name in the final_join.csv to Match the Analysis Code**

- quit shell with `quit()` and rename `final_join` as final_join_old:
    
    hdfs dfs -mv team3/final_join.csv team3/final_join_old.csv
    
- open spark by running the following:
    
    module load python/gcc/3.7.9
    pyspark --deploy-mode client
    
- read in the final_join.csv, rename a column, save to the `team3/etl_code` dir at user home. You can change the path, but remember to adjust the code below:
    
    import pandas as pd
    import numpy as np
    a4 = spark.read.options(header ='True',inferSchema='True',delimiter=',').csv("team3/final_join_old.csv")
    a4 = a4.toPandas()
    a4 = a4.rename(columns={"crime_counts < 0.05": "crime_counts"})
    a4=spark.createDataFrame(a4)
    # output a4 as final_join to 2 locations:
    
    a4.coalesce(1).write.format('com.databricks.spark.csv').options(header='true').save('team3/final_join.csv')
    a4.coalesce(1).write.format('com.databricks.spark.csv').options(header='true').save('team3/etl_code/final_join.csv')
    

## Data Profiling (Post-Joint) - Inspect/Clean data, Calculate descriptive statistics on the joint data

### Profiling on police_short

- quit shell with `quit()` and upload the police_short_profile.py to peel (directory should change to be specific to your computer)
    
    scp profiling_code/police_short_profile.py mc7787@peel.hpc.nyu.edu:/home/mc7787
    
- run:
    
    module load python/gcc/3.7.9
    
    PYTHONSTARTUP=police_short_profile.py pyspark --deploy-mode client 
    

### Profiling for airbnb and business data are included in the cleaning process

### Profiling on joint data

- quit shell with `quit()` and run:
    
    module load python/gcc/3.7.9
    
    PYTHONSTARTUP=descriptive_stats.py pyspark --deploy-mode client
    
- some other data profiling is already done in `business_cleaning.py` and `final_join.py`

## Data Analysis - Spearman Correlation

### Analytics while grouping data by borough:

- quit shell with `quit()` and run:
    
     module load python/gcc/3.7.9
    
     PYTHONSTARTUP=correlation_by_borough.py pyspark --deploy-mode client
    

### Prepare for another analysis and clean final_join.csv again

- quit shell with `quit()` and upload cleaning4.py  to peel home and run, note the file path is different on your computer:
    
    scp etl_code/mc7787MingCao/cleaning4.py mc7787@peel.hpc.nyu.edu:/home/mc7787
    
- Rename 3 columns in team3/etl_code/final_join.csv
    - run:
        
         hdfs dfs -mv team3/etl_code/final_join.csv team3/etl_code/final_join_old.csv
        
    - start spark shell:
        
        module load python/gcc/3.7.9
        pyspark --deploy-mode client
        
    - run:
        
        import pandas as pd
        import numpy as np
        a4 = spark.read.options(header ='True',inferSchema='True',delimiter=',').csv("team3/etl_code/final_join_old.csv")
        a4 = a4.toPandas()
        a4 = a4.rename(columns={'closest crime PdId': 'PdId', 'latitude':'longitude_z29', 'longitude':'longitude_z30'})
        
        a4 = a4.drop(['Location Id'], axis = 1)
        a4=spark.createDataFrame(a4)
        # output a4 as final_join to 2 locations
        a4.coalesce(1).write.format('com.databricks.spark.csv').options(header='true').save('team3/etl_code/final_join.csv') 
        
    - `quit()` spark shell and run:
        
        PYTHONSTARTUP=cleaning4.py pyspark --deploy-mode client
        
- output: `X_price.csv` `X_review.csv`  `y_price_real.csv` `y_review_bi.csv`  in peel home’s team3/etl_code folder
- upload the output csvs to hdfs by running the following:
    
    hdfs dfs -put team3/etl_code/X_price.csv team3/etl_code
    
    hdfs dfs -put team3/etl_code/X_review.csv team3/etl_code
    
    hdfs dfs -put team3/etl_code/y_price_real.csv team3/etl_code
    
    hdfs dfs -put team3/etl_code/y_review_bi.csv team3/etl_code
    

## Data Analysis - Linear Regression, PCA

### PCA random forest analysis:

- upload the acode.py to peel home (please change the directory to make it specific to your computer)
    
    scp ana_code/acode.py mc7787@peel.hpc.nyu.edu:/home/mc7787
    
- add a configuration on hdfs for the [acode.py](http://acode.py) file to work (acode.py file will read from both locations:  `team3/etl_code/final_join.csv`  and `project/final_join.csv`  on hdfs)
    
    hdfs dfs -mkdir project
    
    hdfs dfs -cp team3/etl_code/final_join.csv project/final_join.csv
    
- the following code will take some time to finish:
    
    module load python/gcc/3.7.9
    
    PYTHONSTARTUP=acode.py pyspark --deploy-mode client