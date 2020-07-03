This is a porting of this [Ksql workshop](https://github.com/confluentinc/demo-scene/blob/master/ksql-workshop/ksql-workshop.adoc) to [influxdb](https://www.influxdata.com/products/influxdb-overview/)

Start influxdb 2.0:
`docker pull quay.io/influxdb/influxdb:2.0.0-beta`

Follow the instructions in the ksql workshop to run the docker-compose file with all the services.

For the telegraf configuration see [telegraf.conf](https://github.com/finoalessiopolimi/influxdb_workshop/blob/master/telegraf.conf)

# Query

## 5.3

##### KSQL
`SELECT * FROM ratings;`

##### Flux
```
from(bucket: "from_kafka")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "ratings")
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> drop(columns: ["_start","_stop","host"])
  |> group()
```

##### KSQL
`SELECT * FROM ratings LIMIT 5;`

##### Flux
```
from(bucket: "from_kafka")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "ratings")
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> drop(columns: ["_start","_stop","host"])
  |> group()
  |> sort(columns: ["_time"], desc: false)
  |> limit(n: 5)
```

## 5.4

##### KSQL
`SELECT USER_ID, STARS, CHANNEL, MESSAGE FROM ratings WHERE STARS <3 AND CHANNEL='iOS' LIMIT 3;`

##### Flux
```
from(bucket: "from_kafka")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> drop(columns:["_start","_stop","host"])
  |> filter(fn: (r) => int(v: r.stars) < 3 and r.channel == "iOS")
  |> filter(fn: (r) => r._field == "user_id" or r._field == "message")
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> group()
  |> sort(columns: ["_time"], desc: false)
  |> keep(columns:["user_id","stars","channel","message"])
  |> limit(n:3)

```
## 6
##### KSQL
```
CREATE STREAM POOR_RATINGS AS SELECT * FROM ratings WHERE STARS <3 AND CHANNEL='iOS';
```
##### Flux task
```
option task = {name: "poor_ratings", every: 10s}

from(bucket: "from_kafka")
	|> range(start: -task.every)
	|> filter(fn: (r) =>
		(int(v: r.stars) < 3 and r.channel == "iOS"))
	|> filter(fn: (r) =>
		(r._field == "user_id" or r._field == "route_id" or r._field == "rating_id" or r._field == "message"))
	|> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
	|> group()
	|> drop(columns: ["host"])
	|> to(bucket: "poor_ratings", org: "polimi_usde", fieldFn: (r) =>
		({
			"user_id": r.user_id,
			"route_id": r.route_id,
			"rating_id": r.rating_id,
			"message": r.message,
		}))
```

## 6.3

##### KSQL
`SELECT STARS, CHANNEL, MESSAGE FROM POOR_RATINGS;`

##### Flux
```
from(bucket: "poor_ratings")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._field == "message")
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> group()
  |> keep(columns:["stars","channel","message"])
```

## 6.5

##### Flux json telegraf conf
See [telegraf_json.conf](https://github.com/finoalessiopolimi/influxdb_workshop/blob/master/telegraf_json.conf)

##### Flux
```
from(bucket: "from_kafka_json")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "kafka_consumer")
  |> drop(columns:["_start","_stop","_measurement","host"])
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> group(columns:["_field"])
```

## 9

##### KSQL
```SELECT R.MESSAGE, C.FIRST_NAME, C.LAST_NAME
FROM RATINGS R INNER JOIN CUSTOMERS C
ON R.USER_ID = C.ID
LIMIT 5;
```

##### Flux
```
import "sql"

messages = from(bucket: "from_kafka")
	|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
    |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
    |> group()
    |> drop(columns:["_start","_stop"])
    |> sort(columns: ["_time"], desc: false)
    |> limit(n:5)

db_data = sql.from(
 driverName: "mysql",
 dataSourceName: "root:debezium@tcp(localhost:3306)/demo",
 query:"SELECT * FROM demo.CUSTOMERS ;"
)
  |> rename(columns: {id: "user_id"})


join(tables: {d1: messages, d2: db_data}, on: ["user_id"], method: "inner")
 |> keep(columns: ["first_name","last_name","message"])
```

## 9.1
##### KSQL
```
CREATE STREAM RATINGS_WITH_CUSTOMER_DATA WITH (PARTITIONS=1) AS
SELECT R.RATING_ID, R.CHANNEL, R.STARS, R.MESSAGE,
       C.ID, C.CLUB_STATUS, C.EMAIL,
       C.FIRST_NAME, C.LAST_NAME
FROM RATINGS R
     INNER JOIN CUSTOMERS C
       ON R.USER_ID = C.ID ;
```

##### Flux task
```
import "sql"

option task = {name: "mysql", every: 10s}

messages = from(bucket: "from_kafka")
	|> range(start: -10s, stop: now())
	|> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
	|> group()
	|> drop(columns: ["_start", "_stop"])

db_data = sql.from(driverName: "mysql", dataSourceName: "root:debezium@tcp(localhost:3306)/demo", query: "SELECT * FROM demo.CUSTOMERS;")
	|> rename(columns: {id: "user_id"})

join(tables: {d1: messages, d2: db_data}, on: ["user_id"], method: "inner")
	|> to(bucket: "ratings_with_customer_data", org: "polimi_usde", fieldFn: (r) =>
		({
            "first_name": r.first_name
            "last_name": r.last_name,
            "email": r.email,
            "gender": r.gender,
            "club_status": r.club_status,
            "comments": r.comments,
            "rating_id": r.rating_id,
            "mesage": r.message,
            "rating_id": r.rating_id
		}))
```

##### Flux
```
from(bucket: "ratings_with_customer_data")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "ratings")
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> group()
  |> drop(columns:["_start","_stop","host","_measurement"])
```
## 9.2

##### KSQL
```
SELECT EMAIL, STARS, MESSAGE
FROM RATINGS_WITH_CUSTOMER_DATA
WHERE CLUB_STATUS='platinum'
  AND STARS <3;
  ```

##### Flux
```
from(bucket: "ratings_with_customer_data")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "ratings")
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> filter(fn: (r) => r.club_status == "platinum")
  |> group()
  |> keep(columns:["email","stars","message"])
```

