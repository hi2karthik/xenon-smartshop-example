# xenon-smartshop-example

A simple, distributed application composed of microservices. All services are implemented with [Xenon](https://github.com/vmware/xenon/).

## Build Xenon Library

To build the core Xenon library and install it in the local maven repository
```bash
$ git clone https://github.com/vmware/xenon.git
$ cd xenon
$ mvn clean install -DskipTests
```

## Build and Run dns-service

To build and run `dns-service`, run:
```bash
$ cd dns-service
$ /gradlew build && java -jar build/libs/dns-service-1.0.0-all.jar --port=8002 --id=dnsHost-8002 --sandbox=build/tmp/xenon
```

> The above builds `dns-service` with gradle, and runs a single, standalone Xenon host located at port 8002 (with the id of `dnsHost-8002`). 

## Build and Run product-service

To build and run `product-service`, run:
```bash
cd product-service
./gradlew build && java -jar build/libs/product-service-1.0.0-all.jar --port=8000 --id=productHost-8000 --sandbox=build/tmp/xenon --dnshost=localhost --dnsport=8002
```

> The above builds `product-service` with gradle, and runs a single, standalone Xenon host located at port 8000 (with the id of `productHost-8000`). It also registers with the DNS server at the specified Host and Port.
> To get a fresh Xenon host instance (with no previous persisted state), replace the `gradle build` portion of the command above with `gradle clean build`.

## Build and Run review-service
Similarly (in a separate command window), to build and run `review-service` on port 8001, run:

```bash
cd review-service
./gradlew build && java -jar build/libs/review-service-1.0.0-all.jar --port=8001 --id=reviewHost-8001 --sandbox=build/tmp/xenon --dnshost=localhost --dnsport=8002
```

## Brief introduction to the services (Domain Model)
The domain model for this "smartshop" is admittedly simple, but it does give a good example of how to use Xenon to build, and individually scale microservices that communicate with each other.

A `Product` (defined by [ProductServiceState](./product-service/src/main/java/com/tcurt628/smartshop/product/ProductService.java#L27) ) consists of the following:
* `name` - string
* `description` - string
* `price` - double

A `Review` (defined by [ReviewServiceState](./review-service/src/main/java/com/tcurt628/smartshop/review/ReviewService.java#L35) ) consists of the following:
* `productLink` - string (URI to the product the review is for)
* `author` - string
* `content` - string
* `stars` - int

## Helpful REST calls for testing microservices
Below are some helpful REST calls for interacting with the distributed application.

### product-service API calls

NOTE: The `product` node group and selector is created on startup by [ProductHost.java](./product-service/src/main/java/com/tcurt628/smartshop/product/ProductHost.java#L40)

* `GET` all node groups: `http://localhost:8000/core/node-groups`
* `GET` `product` node group: `http://localhost:8000/core/node-groups/product`
* `GET` `product` node selector: `http://localhost:8000/core/node-selectors/product`
* `GET` all products: `http://localhost:8000/products`
* `POST` to create new product: `http://localhost:8000/products`
```json
{
  "name": "Phillips Hue Lightbulb",
  "description": "Smart, colorful light",
  "price": 59.99
}
```
* `POST` for [QueryTask](https://github.com/vmware/xenon/wiki/Introduction-to-Service-Queries) to find all products: `http://localhost:8000/core/query-tasks`
```json
{
  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
    "query": {
      "term": {
        "propertyName": "documentKind",
        "matchValue": "com:tcurt628:smartshop:product:ProductService:ProductServiceState",
        "matchType": "TERM"
      }
    }
  },
  "indexLink": "/core/document-index"
}
```
* `POST` for [QueryTask](https://github.com/vmware/xenon/wiki/Introduction-to-Service-Queries) to find a product by its id (aka: `documentSelfLink`): `http://localhost:8000/core/query-tasks`
```json
{
  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
    "query": {
      "booleanClauses": [
         {
           "occurance": "MUST_OCCUR",
           "term": {
             "propertyName": "documentKind",
             "matchValue": "com:tcurt628:smartshop:product:ProductService:ProductServiceState",
             "matchType": "TERM"
           }
         },
         {
           "occurance": "MUST_OCCUR",
           "term": {
             "propertyName": "documentSelfLink",
             "matchValue": "/products/1cdc3519-a03f-4373-9686-2ce2f0952a0d",
             "matchType": "TERM"
           }
         }
      ]
    }
  },
  "indexLink": "/core/document-index"
}
```
* `POST` for [QueryTask](https://github.com/vmware/xenon/wiki/Introduction-to-Service-Queries) to find a product by its name: `http://localhost:8000/core/query-tasks`
```json

  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
      "query": {
        "booleanClauses": [
            {
                "occurance": "MUST_OCCUR",
                "term": {
                    "propertyName": "documentKind",
                    "matchValue": "com:tcurt628:smartshop:product:ProductService:ProductServiceState",
                    "matchType": "TERM"
                }
             },
             {
                "occurance": "MUST_OCCUR",
                "term": {
                    "propertyName": "name",
                    "matchValue": "Phillips Hue Lightbulb",
                    "matchType": "TERM"
                }
             }
        ]
      }
  },
  "indexLink": "/core/document-index"
}
```

### review-service API calls

NOTE: The `review` node group and selector is created on startup by [ReviewHost.java](review-service/src/main/java/com/tcurt628/smartshop/review/ReviewHost.java#L41)

* `GET` all node groups: `http://localhost:8001/core/node-groups`
* `GET` `product` node group: `http://localhost:8001/core/node-groups/product`
* `GET` `product` node selector: `http://localhost:8001/core/node-selectors/product`
* `POST` to join `productHost-8000`'s `product` node group as a [OBSERVER](https://github.com/vmware/xenon/wiki/NodeGroupService#node-options)
```json
{
    "kind": "com:vmware:xenon:services:common:NodeGroupService:JoinPeerRequest",
    "memberGroupReference": "http://127.0.0.1:8000/core/node-groups/product",
    "membershipQuorum": 1,
    "localNodeOptions": [ "OBSERVER" ]
}
```
* `GET` `review` node group: `http://localhost:8001/core/node-groups/review`
* `GET` `review` node selector: `http://localhost:8001/core/node-selectors/review`
* `GET` all reviews: `http://localhost:8001/reviews`
* `POST` (to `reviewHost-8001`) for [QueryTask](https://github.com/vmware/xenon/wiki/Introduction-to-Service-Queries) to find all products: `http://localhost:8001/core/query-tasks`
```json
{
  "taskInfo": {
    "isDirect": true
  },
  "querySpec": {
    "query": {
      "term": {
        "propertyName": "documentKind",
        "matchValue": "com:tcurt628:smartshop:product:ProductService:ProductServiceState",
        "matchType": "TERM"
      }
    }
  },
  "indexLink": "/core/document-index"
}
```
* `POST` to create a new review: `http://localhost:8001/reviews`
```json
{
  "stars": 5,
  "author": "tcurtis@vmware.com",
  "content": "Love it",
  "productLink": "/products/1cdc3519-a03f-4373-9686-2ce2f0952a0d"
}`
```
The above tries to create a new `Review`. And it also validates the `productLink` via two ways

