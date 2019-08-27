# Serverless simple web application

This simple application uses these technologies:

* Single Page Application based on Angular 1.4
* CosmosDB database for persistence (non SQL database)
* Function App for hosting REST API and for serving static content
* Azure storage account (blob storage) for our static content

## preparation - deployment of infrastructure

```bash
cd module01
```

**Important** - please customize provided `rc` file in folder `module01`, especially check name of resource group and unique name for our services.

```bash
source rc
```

Now let's prepare our infrastructure

```bash
# create resource group 
az group create --name ${RG} --location ${LOCATION}

# Create cosmosDB
az cosmosdb create --name ${COSMOSDBNAME} --resource-group ${RG}

# Create cosmosDB database
az cosmosdb database create --db-name TestDB \
  --name ${COSMOSDBNAME} --resource-group ${RG}

# Create cosmosDB collection
az cosmosdb collection create --collection-name TestCollection --db-name TestDB \
  --partition-key-path "/pkey" \
  --throughput 400 \
  --name ${COSMOSDBNAME} --resource-group ${RG}

# Create storage account
az storage account create -n ${STORAGENAME} -g ${RG} --sku Standard_LRS

# Create Function App
az functionapp create -n ${FUNCAPP} -g ${RG} -s ${STORAGENAME} --consumption-plan-location ${LOCATION}

# Create container in storage account
az storage container create -n spa --account-name ${STORAGENAME} --public-access container

# Copy SPA files to blob storage container
az storage blob upload-batch --account-name ${STORAGENAME} -d spa -s www-funcapp

```

## Now create logic - functions and static content proxies

### Proxies

Open in azure portal Function App, in the section **Proxies** create these three rules:

* Rule for default document
    * Name: `WebServerDefaults`
    * Route template: `/`
    * Backend URL: `https://YOURSTORAGENAME.blob.core.windows.net/spa/index.html`
* Rule for static content
    * Name: `WebServer`
    * Route template: `{*path}`
    * Backend URL: `https://YOURSTORAGENAME.blob.core.windows.net/spa/{path}`
* Rule for APIs
    * Name: `APIs`
    * Route template: `/api/{*rest}`
    * Backend URL: `https://%WEBSITE_HOSTNAME%/api/{rest}`

### Functions

Now we will create functions for list all items, add item and update item. All functions will be created without security restrictions (no authentication).

#### Create first dummy function

* Runtime stack: `JavaScript`
* Author function quickly in the portal
* Trigger type: `Webhook + API`

#### List of all items

Create new Function with these attributes:

* Click on `Functions` and then click `New function` 
* Click `HTTP trigger`
* Function name: `ToDoList`
* Authorization level: `Anonymous`

In the function settings go to `Integrate` and add output parameter for CosmosDB.

* In the `Inputs` section click `New Input`
* Select `Azure Cosmos DB`
* Install extension for Cosmos DB
* Change Database name to: `TestDB`
* Change Collection name to: `TestCollection`
* Define connection to our existing Cosmos DB
* Save configuration

Now put this source code to code window of our `ToDoList` function.

```javascript
module.exports = function(context, req) {
    context.log('JavaScript HTTP trigger function processed a request.');

    context.res = {
        // status: 200, /* Defaults to 200 */
        body: context.bindings.inputDocument
    };

    context.done();
};
```

#### Add item

Create new Function with these attributes:

* Click on `Functions` and then click `New function` 
* Click `HTTP trigger`
* Function name: `ToDoAdd`
* Authorization level: `Anonymous`

In the function settings go to `Integrate` and add output parameter for CosmosDB.

* In the `Outputs` section click `New Output`
* Select `Azure Cosmos DB`
* Install extension for Cosmos DB
* Change Database name to: `TestDB`
* Change Collection name to: `TestCollection`
* Define connection to our existing Cosmos DB
* Save configuration

Now put this source code to code window of our `ToDoAdd` function.

```javascript
module.exports = function(context, req) {
    context.log('JavaScript HTTP trigger function processed a request.');

    var d = new Date();

    var datestring = d.getDate()  + "-" + (d.getMonth()+1) + "-" + d.getFullYear();

    if (req.body) {
        context.bindings.outputDocument = {
            id: 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
                var r = Math.random() * 16 | 0,
                    v = c == 'x' ? r : r & 0x3 | 0x8;
                return v.toString(16);
            }),
            pkey: datestring,
            created: new Date(),
            updated: new Date(),
            category: req.body.category,
            comment: req.body.comment
        }

        context.res = {
            // status: 200, /* Defaults to 200 */
            body: "Todo item created with id: " + context.bindings.outputDocument.id
        };
    } else {
        context.res = {
            status: 400,
            body: "Insufficient parameters!"
        };
    }
    context.done();
};
```

#### Update item

Create new Function with these attributes:

* Click on `Functions` and then click `New function` 
* Click `HTTP trigger`
* Function name: `ToDoUpdate`
* Authorization level: `Anonymous`

In the function settings go to `Integrate` and add output parameter for CosmosDB.

Input object:
* In the `Inputs` section click `New Input`
* Select `Azure Cosmos DB`
* Install extension for Cosmos DB
* Change Database name to: `TestDB`
* Change Collection name to: `TestCollection`
* Document ID: `{id}`
* Partition key: `{pkey}`
* Define connection to our existing Cosmos DB
* Save configuration

Output object:
* In the `Outputs` section click `New Output`
* Select `Azure Cosmos DB`
* Install extension for Cosmos DB
* Change Database name to: `TestDB`
* Change Collection name to: `TestCollection`
* Define connection to our existing Cosmos DB
* Save configuration

Now put this source code to code window of our `ToDoUpdate` function.

```javascript
module.exports = async function (context, req) {
    context.log('JavaScript HTTP trigger function processed a request.');

    if(context.bindings.inputDocument._etag != req.body._etag){
        context.res = {
            status: 400, /* Defaults to 200 */
            body: "etag is different " + context.bindings.inputDocument._etag + " | " + req.body._etag
        };
    }else{
        context.bindings.outputDocument = req.body;
        context.bindings.outputDocument.updated = new Date();
        context.res = {
            // status: 200, /* Defaults to 200 */
            body: context.bindings.outputDocument
        };
    }

    context.done();
};
```





