
# Overview

This example shows how KSQL can be used to process a stream of click data, aggregate and filter it, and join to information about the users.
Visualisation of the results is provided by Grafana, on top of data streamed to Elasticsearch. 

## Documentation

You can find the documentation for running this example and its accompanying tutorial at [https://docs.confluent.io/platform/current/tutorials/examples/clickstream/docs/index.html](https://docs.confluent.io/platform/current/tutorials/examples/clickstream/docs/index.html?utm_source=github&utm_medium=demo&utm_campaign=ch.examples_type.community_content.clickstream)

# Example steps

## Startup
1. If using Windows, first install Ubuntu LTS from Windows Store:<br/> https://apps.microsoft.com/store/detail/ubuntu-22042-lts/9PN20MSR04DW?hl=en-en&gl=de&rtc=1

2. If git authorization is not yet configured, generate ssh keys, add public key in your account, add git host to list.
3. Clone project form your git repo:
    ```
    git clone git@git.epam.com:<your-name>/m10_kafkabasics_sql_local.git
    ```
4. Get the Jar files for kafka-connect-datagen (source connector) and kafka-connect-elasticsearch (sink connector):
    ```
    docker run -v $PWD/confluent-hub-components:/share/confluent-hub-components confluentinc/ksqldb-server:0.8.0 confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:0.4.0
    docker run -v $PWD/confluent-hub-components:/share/confluent-hub-components confluentinc/ksqldb-server:0.8.0 confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:10.0.2
    ```

   ![sdf](docs/screenshots/1%20install%20datagen.PNG)
   ![sdf](docs/screenshots/2%20install%20elasticsearch.PNG)

5. Launch the tutorial in Docker.
    ```
    docker-compose up -d
    ```

   ![sdf](docs/screenshots/3%20docker%20compose%20up.PNG)
6. After a minute or so, run the docker-compose ps status command to ensure that everything has started correctly.
    ```
    docker-compose ps
    ```
   ![sdf](docs/screenshots/4%20docker%20compose%20ps.PNG)

## Create the Clickstream Data
1. Launch the ksqlDB CLI:
    ```
    docker-compose exec ksqldb-cli ksql http://ksqldb-server:8088
    ```

   ![sdf](docs/screenshots/5%20launch%20of%20ksqlDB.PNG)
2. Ensure the ksqlDB server is ready to receive requests by running the following until it succeeds:
    ```
    show topics;
    ```

   ![sdf](docs/screenshots/6%20show%20topics.PNG)
3. Run the script create-connectors.sql that executes the ksqlDB statements to create three source connectors for generating mock data.
    ```
    RUN SCRIPT '/scripts/create-connectors.sql';
    ```
    ![sdf](docs/screenshots/7%20run%20create%20connectors.PNG)
4. Now the clickstream generator is running, simulating the stream of clicks. Sample the messages in the clickstream topic:
    ```
    print clickstream limit 3;
    ```
    ![sdf](docs/screenshots/8%20clickstream%20sample.PNG)
5. The second data generator running is for the HTTP status codes. Sample the messages in the clickstream_codes topic:
    ```
    print clickstream_codes limit 3;
    ```
   ![sdf](docs/screenshots/9%20http%20codes%20stream%20sample.PNG)
6. The third data generator is for the user information. Sample the messages in the clickstream_users topic:
    ```
    print clickstream_users limit 3;
    ```
   ![sdf](docs/screenshots/10%20user%20stream%20sample.PNG)
7. Go to Confluent Control Center UI at http://localhost:9021 and view the three kafka-connect-datagen source connectors created with the ksqlDB CLI.
   ![sdf](docs/screenshots/11%20connectors%20UI.PNG)

## Load the Streaming Data to ksqlDB
1. Load the statements.sql file that runs the tutorial app.
```
RUN SCRIPT '/scripts/statements.sql';
```
   ![sdf](docs/screenshots/12%20run%20statements%20script.PNG)

## Verify the data
1. Go to Confluent Control Center UI at http://localhost:9021, and view the ksqlDB view Flow.
   ![sdf](docs/screenshots/13%20ksqldb%20view%20.PNG)
2. Verify that data is being streamed through various tables and streams. Query one of the streams CLICKSTREAM:
   ![sdf](docs/screenshots/14%20verify%20data.PNG)

## Load the Clickstream Data in Grafana
1. Set up the required Elasticsearch document mapping template
    ```
    docker-compose exec elasticsearch bash -c '/scripts/elastic-dynamic-template.sh'
    ```
   ![sdf](docs/screenshots/15%20mapping.PNG)

2. Run this command to send the ksqlDB tables to Elasticsearch and Grafana:
    ```
    docker-compose exec ksqldb-server bash -c '/scripts/ksql-tables-to-grafana.sh'
    ```
   ![sdf](docs/screenshots/16%20send%20ksqlDB%20tables%20to%20ElSearch%20and%20Graphana.PNG)
   
3. Load the dashboard into Grafana.
    ```
    docker-compose exec grafana bash -c '/scripts/clickstream-analysis-dashboard.sh'
    ```
   ![sdf](docs/screenshots/17%20load%20dashboard%20into%20grafana.PNG)
   
4. Navigate to the Grafana dashboard at http://localhost:3000. Enter the username and password as user and user. Then navigate to the Clickstream Analysis Dashboard.
   ![sdf](docs/screenshots/19%20grafana%20dashboard.PNG)
5. In the Confluent Control Center UI at http://localhost:9021, again view the running connectors. The three kafka-connect-datagen source connectors were created with the ksqlDB CLI, and the seven Elasticsearch sink connectors were created with the ksqlDB REST API.
   ![sdf](docs/screenshots/20%20source%20and%20sink%20connectors.PNG)

## Sessionize the data

To generate the session data execute the following statement from the examples/clickstream directory:

```
./sessionize-data.sh
```

The script will issue some statements to the console about where it is in the process.

# Task metrics

1. General website analytics, such as hit count and visitors (create graphic from existing panel)
   ![sdf](docs/screenshots/21%201%20hit%20count%20&%20visitors.png)
2. Bandwidth use
   - First create table in ksqldb
    ```
    CREATE TABLE BYTES_PER_MIN AS 
      select  USERID, SUM(BYTES), WINDOWSTART as EVENT_TS 
      FROM USER_CLICKSTREAM window TUMBLING (size 60 second)
      GROUP BY USERID;
    ```
   - Then register it in Grafana
   ```
    docker-compose exec ksqldb-server bash -c '/scripts/ksql-connect-es-grafana.sh "BYTES_PER_MIN"'
    ```
   - Then create a panel
   ![sdf](docs/screenshots/21%202%20bandwith.png)
3. Mapping user-IP addresses to actual users and their location (check existing dashboard)
   ![sdf](docs/screenshots/21%203%20ip%20to%20address.PNG)
4. ...
5. Error-code occurrence and enrichment (check existing dashboard)
   ![sdf](docs/screenshots/21%205%20errors.PNG)
6. Sessionization to track user-sessions and understand behavior (such as per-user-session-bandwidth, per-user-session-hits etc) 
(check after running `./sessionize-data.sh` script)
   ![sdf](docs/screenshots/21%206%20sessionizing.PNG)