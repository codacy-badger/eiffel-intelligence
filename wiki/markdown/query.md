# Query

|Method|Endpoint             |
|------|---------------------|
|GET   |/queryAggregatedObject|
|POST   |/query|
|GET   |/queryMissedNotifications|

## Perform query on created aggregated object
    GET /queryAggregatedObject

**Query parameters**:
ID=\<documentID\>

Examples of this endpoint using curl

    curl -X GET -H "Content-type: application/json"  http://localhost:8090/queryAggregatedObject?ID=6acc3c87-75e0-4b6d-88f5-b1a5d4e62b43

## Perform freestyle query on created aggregated object
It is possible to query for documents using freestyle queries. These freestyle
queries are plain Mongo DB queries, you can read more about that [here](https://docs.mongodb.com/manual/tutorial/query-documents/).

    POST /query

**Body (application/json)**:

    {
      "criteria": <MongoDB query>
      "options": <MongoDB query> // This one is optional
      "filter": <JMESPath expression> // This one is optional
    }

Examples of this endpoint using curl

    curl -X POST -H "Content-type: application/json"  --data @Body.json http://localhost:34911/query

Examples of criterias:

    // This returns all objects where the object.testCaseExecutions.outcome.id is"TC5"
    // or object.testCaseExecutions.outcome.id is "TC6" and "object.identity"
    // is "pkg:maven/com.mycompany.myproduct/sub-system@1.1.0".

    {
      "criteria": {
        "$or": [
          {
            "object.testCaseExecutions.outcome.id":"TC5"
          }, {
            "object.testCaseExecutions.outcome.id":"TC6"
          }
        ],
        "object.identity":"pkg:maven/com.mycompany.myproduct/sub-system@1.1.0"
      }
    }


    // This returns all objects where TC5 succeeded in
    // the array testCaseExecutions and object.identity is "pkg:maven/com.mycompany.myproduct/sub-system@1.1.0".
    // Options section is optional.

    {
      "criteria": {
        "object.testCaseExecutions": {
          "$elemMatch": {
            "outcome.conclusion": "SUCCESSFUL",
            "outcome.id": "TC5"
          }
        }
      },
      "options": {
        "object.identity":"pkg:maven/com.mycompany.myproduct/sub-system@1.1.0"
      }
    }


    // This returns all objects where TC5 succeeded
    // and are related to JIRA ticket JIRA-1234
    {
        "criteria": {
             "object.testCaseExecutions": {
                 "$elemMatch": {
                     "outcome.conclusion": "SUCCESSFUL",
                     "outcome.id": "TC5"
                 }
             },
             "$where" : "function() {
                 var searchKey = 'id';
                 var value = 'JIRA-1234';
                 return searchInObj(obj);
                 function searchInObj(obj){
                     for (var k in obj){
                         if (obj[k] == value && k == searchKey) {
                             return true;
                         }

                         if (isObject(obj[k]) && obj[k] !== null) {
                             if (searchInObj(obj[k])) {
                                 return true;
                             }
                         }
                     }
                     return false;
                 }
             }"
        }
    }

NOTE: It should be noted that "object" is a key word.  It is replaced
dynamically in the code, and is configured in the application.properties file with
the search.query.prefix option. e.g., in string "object.testCaseExecutions.outcome.id",
substring "object" is a key word.

## Example of freestyle query that returns all aggregated objects
By using a query that contains only empty "criteria" it is possible to return
all aggregated objects from the database. The aggregated objects will be
returned from specific collection (which name is defined by property
aggregated.collection.name) that is stored in specific database (which name is
defined by property spring.data.mongodb.database). Read more about the
different properties in [application's properties](https://github.com/Ericsson/eiffel-intelligence/blob/master/src/main/resources/application.properties)
or in the [documentation](https://github.com/eiffel-community/eiffel-intelligence/blob/master/wiki/markdown/configuration.md).

Example:

    {
      "criteria": {}
    }


## Query an aggregated object and filter it with specific key
It is possible to filter the object and return only values with specific key or
path. To do this, it is required to add filter condition to the json body. The
parameter of filter condition is a JMESPath expression, you can read more about
that [here](http://jmespath.org/tutorial.html#pipe-expressions).

Example:

    // This match an object in the same way as in the previous example. And then filter
    // it and returns eventId that has a path "aggregatedObject.publications[0].eventId".
    // aggregatedObject is the name of the aggregated object in the database.

    {
      "criteria": {
        "object.testCaseExecutions": {
          "$elemMatch": {
            "outcome.conclusion": "SUCCESSFUL",
            "outcome.id": "TC5"
          }
        }
      },
      "options": {
        "object.identity":"pkg:maven/com.mycompany.myproduct/sub-system@1.1.0"
      },
      "filter" : "aggregatedObject.publications[0].eventId"
    }


To filter with only the key or the partial path, it is required to use
"incomplete_path_filter(@, 'some key')".

Example:

    // This finds all objects where identity is "pkg:maven/com.mycompany.myproduct/sub-system@1.1.0".
    // Then it filters those objects and returns all values that has "svnIdentifier" as a key.

    {
      "criteria": {
         "object.identity":"pkg:maven/com.mycompany.myproduct/sub-system@1.1.0"
      },
      "filter" : "incomplete_path_filter(@, 'svnIdentifier')"
    }

As the filter functionality takes a plain jmespath expression it can be a more
complex expression as well. In the case below we retrieve all aggregated objects
that contains a certain commit using the criteria(MongoDB query) and then filter
out which confidence levels the artifact has succeeded on. NOTE that the Mongo
DB query as well as the jmespath expression needs to be on the same line as json
cannot handle multilines inside a string. Although some tools such as curl will
automatically minimize.

Example

    {
        "criteria": {
            "$where": "function(){ var searchKey = 'id'; var value = 'JIRA-1234'; return searchInObj(obj); function searchInObj(obj){ for (var k in obj){ if (obj[k] == value && k == searchKey) { return true;  } if (isObject(obj[k]) && obj[k] !== null) { if (searchInObj(obj[k])) { return true;}}} return false; }}"
        },
        "filter": "{id: aggregatedObject.id, artifactIdentity:aggregatedObject.identity, confidenceLevels:aggregatedObject.confidenceLevels[?value=='SUCCESS'].{name: name, value: value}}"
    }

## Query missed notifications
    GET /queryMissedNotifications

**Query parameters**:
SubscriptionName=<Subscription Name/Subscription ID>

Examples of this endpoint using curl

    curl -X GET -H "Content-type: application/json" localhost:39835/queryMissedNotifications?SubscriptionName=Subscription_1

