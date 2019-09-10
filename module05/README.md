# Serverless data processing 

This sample is based on original repository there: https://github.com/valda-z/demo-azure-bi-analytic

Demonstration of event processing and handling which uses these technologies:

* Azure Function App - Webhook API for data collection
* Event Hub for data collection
* Stream analytics for data processing
* PowerBI for data presentation
* Azure storage account (table storage for data persistence)

## prepare Event Hub

In our experimental resource group from module01 we will create new Event Hub resource:

* `Create a resource` in portal and select `Event Hub`
    * define unique name for event hub
    * use `Basic` pricing tier
    * select your existing resource group
    * select your favorite region
* goto your `Event Hubs Namespace` and create there new Event Hub - there is button `+ Event Hub`
    * use name `demo`

## Create Azure functions

We will use our Function App from module01, please go to Azure Function App in azure portal:

* Create new function `yes`
    * Type: `Http trigger`
    * Name: `yes`
    * Authorization level: `Anonymous`
    * Go to `Integrate` option for function
        * Add new output
        * Select type `Azure Event Hubs`
        * Please install site extension if necessary
        * Define `Event Hub connection` to your Event Hub
        * Change Event Hub name to `demo`
        * Save changes
    * Go to function body and paste there this JavaScript code

```javascript
module.exports = function (context, req) {
    var timeStamp = new Date().toISOString();
    context.bindings.outputEventHubMessage = {
        id: 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
                var r = Math.random() * 16 | 0,
                    v = c == 'x' ? r : r & 0x3 | 0x8;
                return v.toString(16);
            }),
        created: new Date(),
        vote: 'YES'
    };
    context.done();
};
```

* Create new function `no`
    * Type: `Http trigger`
    * Name: `no`
    * Authorization level: `Anonymous`
    * Go to `Integrate` option for function
        * Add new output
        * Select type `Azure Event Hubs`
        * Define `Event Hub connection` to your Event Hub
        * Change Event Hub name to `demo`
        * Save changes
    * Go to function body and paste there this JavaScript code

```javascript
module.exports = function (context, req) {
    var timeStamp = new Date().toISOString();
    context.bindings.outputEventHubMessage = {
        id: 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
                var r = Math.random() * 16 | 0,
                    v = c == 'x' ? r : r & 0x3 | 0x8;
                return v.toString(16);
            }),
        created: new Date(),
        vote: 'NO'
    };
    context.done();
};
```

Now you can test both functions directly from portal if they are working and sending messages to Event Hub, eventualy you can check in Azure portal that Event Hub is receiving data (on statistics charts).

## Create Stream Analytics for data processing

In Azure portal create new resource `Stream Analytics job`, use your favorite region and resource group used in module01.

* Define Job name: `demo`, use your resource group and your favorite region
* after creation open new resource `demo` (Stream Analytics job)
* Go to `Inputs` - there we will define input stream for processing
    * `+ Add Stream input`, select type `Event hub`
        * Input alias: `fromeventhub`
        * Select your Event Hub name
* Go to `Outputs` - there we will create output data consumers
    * `+ Add` to define output
        * Select type `PowerBi`
        * Output alias: `topowerbi`
        * Dataset name: `AcademyTest`
        * Table name: `AcademyTestTable`
        * Authorize connection to your PowerBi ..
    * `+ Add` to define output
        * Select type `Table storage`
        * Output alias: `totablestorage`
        * Select your blob storage from resource group from module01
        * Table name - create new: `mytable`
        * Partition key: `vote`
        * Row key: `id`
        * Batch size: 10
* Go to `Query` - define processing query
```sql
SELECT
    id,
    CAST(DATEADD(hour,1,created) as datetime) as Time,
    DATENAME (year, DATEADD(hour,1,created)) as Year,
    DATENAME (month, DATEADD(hour,1,created)) as Month,
    DATENAME (day, DATEADD(hour,1,created)) as Day,
    DATENAME (weekday, DATEADD(hour,1,created)) as WeekDay,
    DATENAME (hour, DATEADD(hour,1,created)) as Hour,
    DATENAME (minute, DATEADD(hour,1,created)) as Minute,
    UPPER(vote) as vote,
    CASE WHEN UPPER(vote)='YES' THEN 1 ELSE 0 END as voteyes,
    CASE WHEN UPPER(vote)='NO' THEN 1 ELSE 0 END as voteno
INTO
    [totablestorage]
FROM
    [fromeventhub] TIMESTAMP BY created

SELECT
    System.TimeStamp AS WindowEnd,
    COUNT(*) as person,
    SUM(CASE WHEN UPPER(vote)='YES' THEN 1 ELSE 0 END) as votesyes,
    SUM(CASE WHEN UPPER(vote)='NO' THEN 1 ELSE 0 END) as votesno
INTO
    [topowerbi]
FROM
    [fromeventhub] TIMESTAMP BY created
GROUP BY TumblingWindow(Duration(second, 5))  
```

* Go to `Overview` and start processing engine
    * `Start` - select from `Now`

## Test and consume data

Now you can call few times URL for `yes` and `no` APIs, if data are correctly processed correctly than you can see new entries in Storage Table in storage (check with storage explorer in Azure portal).

And finally you can create new live dashboard in PowerBi which will consume our new dataset produced by stream analytics.
        
