# URL Shortener Service

## Summary
My service is a self-hostable link shortening web application.  
The functionality is provided via REST API. You can:  
 * Create & Redirect short urls 
 * Query statistics about clicks, browsers, and referrers.
 
Application is written in Java using Cassandra as its primary database.
## How to run

### Requirements

* Java SDK 8
* Docker v18.09.4 & Docker Compose v1.21.0

### Build Application
Extract files from **url-shortener-service.zip**:
   ```sybase
    unzip url-shortener-service.zip 
  ```
  
Inside the newly created **url-shortener-service** directory build the target:
```sybase
./mvnw clean package
```

### Run Application
```sybase
docker-compose up -d
```
> You need to wait some time while CassandraDB is starting.

To check whether the app is running type the following command:

```sybase
docker-compose ps
```
If everything is OK, you will see this:
```sybase
cassandra               docker-entrypoint.sh cassa ...   Up (healthy)   7000/tcp, 7001/tcp, 7199/tcp, 0.0.0.0:9042->9042/tcp, 9160/tcp
cassandra-load-schema   docker-entrypoint.sh /bin/ ...   Exit 0                                                                       
urlshortener            java -cp app:app/lib/* com ...   Up             0.0.0.0:8080->8080/tcp  
```

### Stop Application

#### Stop and remove all persistent data (e.g. short links)
Fire the following command in order to stop the application:
```sybase
docker-compose down --volumes
```
#### Stop without removing persistent data
```sybase
docker-compose down
```
> **cassandra-load-schema** service will have **Exit 2** state if you stop the app
without data deletion and then run the app again. But that's fine, since 
all the tables have already been created.

## Usage

### Create A Short Link

```
POST http://localhost:8080/api/v1/urls/
Body: '{"longUrl":"<your long URL"}'
```

Example Request:
```sybase
curl --request POST \
  'http://localhost:8080/api/v1/urls/' \
  --header 'Accept: application/json' \
  --header 'Content-Type: application/json' \
  --data '{"longUrl":"https://yandex.ru"}' \
```

Example Response:
```sybase
{
   "id" : "dO7xaQb3A",
   "longUrl" : "https://yandex.ru",
   "shortUrl" : "http://localhost:8080/dO7xaQb3A",
   "created" : "2019-05-21T01:55:57.765Z"
}
```

### Get Information About A Short Url



```
GET http://localhost:8080/api/v1/urls/{short-url-id}
```

Example Request:
```sybase
curl http://localhost:8080/api/v1/urls/dO7xaQb3A
```

Example Response:
```sybase
{
   "id" : "dO7xaQb3A",
   "longUrl" : "https://yandex.ru",
   "shortUrl" : "http://localhost:8080/dO7xaQb3A",
   "created" : "2019-05-21T01:55:57.765Z"
}
```

### Short Link Redirection

Example Request:
```sybase
http://localhost:8080/dO7xaQb3A
```

### Query Statistics About Clicks

#### Total number of clicks
```sybase
http://localhost:8080/api/v1/urls/dO7xaQb3A/clicks
```
  
```sybase
{
   "clicks" : 3
}
```

#### Clicks by web browsers:
```sybase
curl http://localhost:8080/api/v1/urls/dO7xaQb3A/browsers
```

```sybase
[
   {
      "browser" : "Chrome",
      "clicks" : 1
   },
   {
      "browser" : "Firefox",
      "clicks" : 2
   }
]
```

#### Clicks by referrers:
```sybase
curl http://localhost:8080/api/v1/urls/dO7xaQb3A/referrers
```

```sybase
[
   {
      "referrer" : "https://developer.mozilla.org/en-US/docs/",
      "clicks" : 1
   },
   {
      "clicks" : 1,
      "referrer" : "https://stackoverflow.com/questions/50340169/"
   }
]
```

## Internals & Future Enhancements

For short link generation I use a distributed ID generator.
I am trying to employ the strategy when a separate integer range is used for each node. 
For this scenario I have implemented a generator similar to Twitter's [Snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)
service for generating unique numbers. My current implementation 
has a limitation: you have to explicitly provide a node id using environment variables. 
Other approaches could be, for example, taking a MAC address as the node id.
This is OK since apps today are usually managed by k8s and each pod has a unique MAC address.
Another option would be to go with an enterprise solution like HazelCast Id Generator

The generated number is then converted to a Base 62 string. For my alphabet I decided to use letters [A-Za-z] and numbers [0-9].
Overall, It gives me 8-10 character length strings. 

Taking into account scalability issues, I decided to go with Cassandra due to 
its capability to work under high load and accommodate many writes. On the other hand, to further
implement short link statistics I will need an analytics engine like Apache Spark to perform efficient aggregations
(e.g. clicks by week, month, etc), since Cassandra initially is not a good fit for this. Generally speaking, 
I want to spend more time for tuning Casandra (e.g. its caching layer). 

Statistics in my application is updated using synchronous calls which are not safe in a case of service failure.
In future I will need to convert those calls to events and have some kind of a message queue to enable event-driven 
communication. 

I want to separate link redirection from other API since redirection is a different thing.
For example, we could move it to another service, At the time we start creating multiple services it's a good time 
to think about creating a single entry point i.e. API gateway, for example, to hide our service decomposition.

Finally, I want to add that the url shortener service needs security: API keys, OAuth, user accounts, etc...
