# aws-lambda-aurora

**Data Modeling**\
The "Orders" dataset includes information for different types of objects.\
Since an order can have multiple products and different customers can place each order, I decided to transform the dataset to a relational data model and use a SQL database as a data storage. Using a SQL database will save a large amount of space for this scenario than a NoSQL solution.

We will have to create four tables.
- customers
- orders
- order items
- products

**Customers Table:**\
The customers table stores customer information and has the customer_id as a primary 
.\
**Orders Table:**\
The orders table stores information about the order (i.e. order_ref) and has the order_id as a primary key and customer_id as a foreign key.\
**Orders Item Table:**\
The order_items table stores the line items of the order and has a composite primary key of order_id and order_item_id. The order_items table has product_id as a foreign key.\
**Products Table:**\
The products table stores information about the products (i.e. product_name, price) and has the product_id as a primary key.

**Data Model Image:**\
<img src="https://github.com/dschizas/aws-lambda-aurora-draft/blob/main/other/DataModel.png" height="500">

**Tech Stack**\
The API uses various AWS resources to deliver a scalable, highly available, idempotent and secure solution. \
Core architecture includes the following:
- Amazon API Gateway
- AWS Lambda with Node.js runtime and GraphQL
- Amazon ElastiCache
- Amazon RDS Proxy
- Amazon Aurora
<img src="https://github.com/dschizas/aws-lambda-aurora-draft/blob/main/other/Aurora.png">

**Development**\
As part of the API development I will use Javascript Lambdas and will follow the TDD. The primary focus will be on the unit and integration tests.  The aim here is to write the smallest test possible, and as the API codebase grows, avoid using a fake test double and use a stub or a mock as a last resort.

**Idempotency**\
Since the application does not update any record in the data storage, I am not worried about looking at a locking strategy to avoid concurrent updates. 

**Security**\
The GraphQL-Lambda-Aurora example offers the client the option of accessing the public API and the relevant secured resources. As part of the security of this example, we can use API Gateway Lambda authorisers for authorisation and restrict who can call an operation.\
The Public API provides access to secured resources. The first line of defence is authorisation - restricting who can call an operation in the GraphQL Schema.\
API Gateway supports multiple mechanisms for controlling and managing access to our API. In this example, I used AWS Lambda authorizers, but another option can be Resource policies, IAM tags, Cognito users pool, to name a few.\
To protect against tampering, the API will use the HTTPS request-response protocol to encode and transport information across the client and the application.\
I will also make sure that Aurora cluster's encryption is enabled.\
The AWS services support different types of security and my goal is to ensure that we have an end-to-end secure design. 

**Database**\
I will use Amazon Aurora since the data model requires a SQL database, and we are interested in supporting read queries. Aurora enables the data storage to withstand a read-intensive workload while providing a natively Highly Available (HA) solution. At the same time, Aurora delivers better performance compared to the other Relational Database Service. 

**RDS Proxy**
To increase the application's availability and protect from outages and failures, we can support database replicas and use RDS Proxy to connect to a new database if needed automatically. RDS proxy maintains the connections to the database instances and can help to reduce the stress on the resources when a new connection occurs.

**ElastiCache**\
In this example, I am looking to build a solution that reads data from an RDS. If we start hitting the max_connections limit and not get the full benefit of the RDS Proxy based on the number of requests we are receiving, we can consider using an ElastiCache. ElastiCache aims to support the calls from the lambda functions and to decrease the database workload. ElastiCache can scale independently but can be expensive, and I recommend using it only if required while considering its configuration (i.e. TTL policies).
I am not worried about persistency for this cache implementation since I won't use it to store values that don't exist on Aurora.

**API Gateway REST API**\
I decided to use API Gateway REST API instead of AppSync for the GraphQL Lambda since the former supports throttling, it has higher requests per second limit and supports private endpoints.\
Similarly, we can implement a rate-limiter and prevent our API from being overwhelmed by too many requests by using the API Gateway's throttling limit or using a queue filler like kinesis streams, Redis or SQS.

**AWS Lambda**\
The GraphQL-Lambda implementation requires the following four files. Please note that most of the code is written in pseudocode format.
- GraphQL API schema
- The handler.js file which includes the database queries and the client implementation
- The serverless.yml file

GraphQL Types:
```
type Customer {
    customerId: ID!
}

type Order {
    orderId: ID!
    orderRef: Int!
    customers: [Customer!]!
}

type OrderItem {
    orders: [Order!]!
    orderItemId: ID!
    products: [Product!]!
}

type Product {
    productId: ID!
    productName: String!
    price: Float!
}
```
GraphQL Query:
```
type Query {
  Product(productId: ID!) {
    productName!
    price!
  }
}

type Query {
  OrderItem(order: ID!) {
    Product!
    orderItemId!
  }
}

type Query {
  Order(customerId: ID!) {
    Customer!
    orderRef!
  }
}
```
GraphQL Schema:
We can skip the definition of mutation operations since this pseudocode example aims to only query the database.
```
schema {
	query: Query
}
```
Now we now need to define the handler.js
```
const { GraphQLServerLambda } = require("graphql-yoga");
var fs = require("fs")
const Client = require('serverless-mysql')
const typeDefs = fs.readFileSync("SchemaFile").toString('utf-8');

var client = Client({
    config: {
        // host, database, user, password
    }
})

const resolvers = {
    Query: {
        getProduct: async (client, productId) => {
          let product = await client.query(`select productId, productName, productPrice from users where productId = ? `, [productId])
          // if(product.length == 0) return null;
          return product;
        }
    },
};

const lambda = new GraphQLServerLambda({
    typeDefs,
    resolvers
});

exports.server = lambda.graphqlHandler;
```
Finally we need to define the serverless.yml.\
```
provider:
  name: aws
  region: us-east-1
  stage: dev
  memorySize: 256
  runtime: nodejs8.10
  role: LambdaRole
  environment:
    AURORA_HOST: ${self:custom.AURORA.HOST}
    AURORA_PORT: ${self:custom.AURORA.PORT}
    DB_NAME: ${self:custom.DB_NAME}
    USERNAME: ${self:custom.USERNAME}
    PASSWORD: ${self:custom.PASSWORD}
custom:
  DB_NAME: graphql
  USERNAME: master
  PASSWORD: password
  AURORA:
    HOST:
      Fn::GetAtt: [AuroraRDSCluster, Endpoint.Address]
    PORT:
      Fn::GetAtt: [AuroraRDSCluster, Endpoint.Port]
    VPC_CIDR: 10
plugins:
  - serverless-pseudo-parameters
resources:
  Resources:
    AuroraRDSInstance:
      DependsOn: ServerlessVPCGA
      Type: AWS::RDS::DBInstance
      Properties:
  	DBInstanceClass: db.t2.small
  	DBSubnetGroupName:
	  Ref: ServerlessSubnetGroup
  	Engine: aurora
  	EngineVersion: "5.6"
  	PubliclyAccessible: true
  	DBParameterGroupName:
    	  Ref: AuroraRDSInstanceParameter
  	DBClusterIdentifier:
    	  Ref: AuroraRDSCluster
functions:
  graphql:
    handler: handler.server
    events:
      - http:
          path: /
          method: post
          cors: true
  playground:
    handler: handler.playground
    events:
      - http:
          path: /
          method: get
          cors: true
```
**Lambdas performance and cold starts**\
When I first execute my request to my endpoint, I noticed an additional latency of ~800ms is added on top of the standard execution time of my lambda function. The initialisation of the Lambda container causes this latency overhead.\
Cloudwatch logs:
```
REPORT RequestId: 06dc303b-6189-4553-a4cb-aec72a3c7886	Duration: 151.77 ms	Billed Duration: 152 ms	Memory Size: 1024 MB	Max Memory Used: 93 MB	Init Duration: 1021.63 ms	
```
```
2021-05-23T16:04:53.456Z	934fcedd-c56f-4de9-9415-cd0295939d1b	INFO	{ Item:
   { orderRef: '123123',
     price: 19.99,
     orderId: 52,
     productId: 'notebook',
     productName: 'notebook_red',
     customerId: 4567 } }
REPORT RequestId: 934fcedd-c56f-4de9-9415-cd0295939d1b	Duration: 26.41 ms	Billed Duration: 27 ms	Memory Size: 1024 MB	Max Memory Used: 93 MB	
```
To solve this issue we can use [serverless-plugin-warmup](https://www.npmjs.com/package/serverless-plugin-warmup) on our Lambdas.

**Considerations**\
When building distributed systems, the goal is usually to make them resilient, elastic and scalable.
In this example, I described the "heavy read" design pattern.\
Depending on existing tooling, I will make sure that the application has the following additional services that can help maintain, protect and scale the development of the solution.
- Monitoring: An option would be to use Cloudwatch (application and infrastructure monitoring) and Cloudtrail (user's activity and API usage)
- Infrastructure as a Code: To have better visibility, stability and scalability of the infrastructure, we can use CloudFormation or Terraform 
- Centralised Key Management: We can use KMS for a fully managed, encrypted key management system







