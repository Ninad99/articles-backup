## Introduction

I've started a new project 🎉

I'll build an opinionated and abstracted version of SQS as a Service. 

The idea is to offer a really easy and still customizable solution for message queues. The target group is developers developing on Vercel, Netlify, or a similar service who probably don't have an AWS Account. 

## Competitors

- [AWS itself](https://aws.amazon.com/sqs/) -> Too complicated to set up (that is the whole point)
- [Quirrel](https://quirrel.dev/) -> Acquired by Netlify and stopped for the public (but open source)
- [Zeplo](https://zeplo.io) -> Great product but too less customizable for me

Zeplo however is a really valid product and a cool alternative. I am not sure if I can beat it because we have kind of the same target group. But I think more about an abstracted version of SQS but I still want to make it accessible and customizable for SQS power users.

## What is SQS?


![Simple Queue Service Icon](https://cdn.hashnode.com/res/hashnode/image/upload/v1649050207853/3gvflhIWw.png)

First of all, what is SQS?

SQS is the Simple Queue Service and it was the first service in AWS. It is a fully managed message queue and allows you to send messages to this queue. Consumers can poll messages and work on that. Consumers are often Lambda Functions or Instances such as EC2 or ECS Containers. 

Over the years I saw that most of the event-driven and serverless use-cases are simply an SQS queue connected with a lambda function. Since systems get more asynchronous and event-driven it is a great design and a great service for that.

Some service (e.g. a Vercel Serverless Function) sends an event to SQS. SQS passes it to Lambda, and Lambda executes some business logic. This is the typical pattern.

## The Problem to Solve

**The Problem is AWS itself**

I identified three main problems:
1. Overhead
2. Separating business logic
3. Understanding SQS

### Overhead for Creating a Simple Queue

Working in an AWS environment means there are certain prerequisites. I saw this problem multiple times while either onboarding new developers to an existing project or teaching them how to implement such use cases. If the only goal is a message queue and no AWS Account exists you need to do (at least) the following things:

- Create an AWS Account
- Set up a credit card with your AWS Account
- Understand IAM (root user, MFA, dedicated users, roles, policies)
- Named profiles locally
- Create budget alerts
- Create an SQS queue
- Create a lambda function
- Orchestrate all of that somehow in code (CDK, Terraform Pulumi) 
- Understand SQS (long polling, visibility timeout, FIFO, idempotency)

These are way too many steps for a simple thing like sending messages to a queue. That is why I wanted to build a service for abstracting this away.

Especially for users from Vercel (Zeit) or Netlify, it is super easy to deploy your website and build APIs and serverless functions without even having an AWS Account. They don't even need an AWS account most of the time. 

### Separating Business Logic

Another problem comes with separating the business logic. If you have a Vercel account you can simply have a serverless function in `pages/api/executor.ts` that can work on your business logic. But if you implement it with SQS -> Lambda you have a separate system and a separate codebase. 

A simple message queue could allow them to send messages from their various backend APIs, queue them, and let another serverless function work on that. 

There is no need for an additional AWS Account.

### SQS is Hard

If you need a 1x1 thread to build a simple message queue it is probably a bit too much.

%[https://twitter.com/tpschmidt_/status/1455928446484455431]

SQS is not super straightforward to learn. I struggled a lot with understanding many things and setting good default parameters. For most of the use-cases, you don't need to be an SQS pro just for a simple queue. But without studying it you almost cannot use SQS properly.

You need to learn many terms like:
- Short polling vs. long polling
- Dead letter queues
- Redrives
- Visibility timeouts
- ... and much more

### Time to Touch Business Logic


![Time till you touch business logic](https://cdn.hashnode.com/res/hashnode/image/upload/v1649050737216/dnTQpkIZ9.png)

A good indicator for me to see that there is an opportunity for making things easier for developers is to take a look at the time it takes someone to touch the business logic. With AWS it takes a lot of time till you reach the point of actually developing your logic in Lambda. With my proposed solution, you simply create a queue and build your business logic in a second. 

That's why I build an opinionated and abstracted version of SQS.



## Prototype

I wanted to start out with a simple prototype to prove if it is even possible to build what I want to build. I gave myself a time restriction of **one weekend** for that. 

The prototype should show that it is possible to have a simple use case like that:


![Prototype Use Case](https://cdn.hashnode.com/res/hashnode/image/upload/v1649051012419/sjK7F6Prs.png)

1. User creates a queue with good default parameters
2. User sends a request to the queue including parameters like the target and my system queues the request and forwards it to the target


The goals for the weekend were to prove that this is technically possible.

## The AWS Architecture

My initial ideas were prototyped on a whiteboard. Sorry for my handwriting.

![Whiteboard Architecture](https://cdn.hashnode.com/res/hashnode/image/upload/v1648991311868/-AfVxYGTo.jpg)

The two major flows that need to be implemented are:
1. Create a queue
2. Receive messages, queue them, and forward them

Let's go through both flows in detail


### Creating a Queue

![Creating a Queue](https://cdn.hashnode.com/res/hashnode/image/upload/v1649051287348/Wu9cesbFT.png)

The following services are involved here:
- AppSync - GraphQL API
- SQS - Actual Queue
- Lambda - Mutation for creating the queue & Executing the `EventSourceMapping`
- DynamoDB - Saving Metadata


**AppSync**

First of all, we've got AppSync which is a fully managed GraphQL API. In AppSync I've got the following schema. I've included only the important parts for the prototype:

```graphql
type Queue {
  id: ID!
  name: String!
  user: User!
  sqsUrl: String!
  arn: String
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type Request {
  id: ID!
  queue: Queue!
  body: String!
  status: String!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type Response {
  id: ID!
  request: Request!
  body: String!
  status: String!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type Mutation {
  createQueue(input: CreateQueueInput!): Queue!
}

input CreateQueueInput {
  name: String!
  userId: ID!
}

```

The main part of the prototype is the type `Queue` and the mutation `createQueue`. Only for the mutation `createQueue` a lambda function is used as a resolver because more business logic is needed for that. All others are resolved with VTL. 


**Lambda**
Calling the mutation `createQueue` calls a lambda function.


The lambda function is doing 3 things:

```ts
// 1. Create queue in SQS with default params

// 2. Add lambda as EventSourceMapping

// 3. Add Metadata to DynamoDB
```

The node.js SDK from AWS can do all three things. Every time a user creates a queue these three things are happening.

Creating a queue is now as easy as that:

```graphql
  createQueue(input: {name: "q1", userId: "123"}) {
    id
  }
```

After that we'll have the following:

- Queue with a random name
- Item in DynamoDB with the URL, arn, user, and name
- `EventSourceMapping` to the lambda function

### Sending a Request

The second flow is sending a request.

![Sending a Request to the Queue](https://cdn.hashnode.com/res/hashnode/image/upload/v1648990281982/ln_n9pXs_.png)

This time these services are included:
- API Gateway - REST API for user requests
- DynamoDB - Getting the correct queue
- Lambda
    - Handling user requests 
    - Executing the request
- SQS - Queue from the User

#### API Gateway

For the REST API I use the service API Gateway. With API Gateway I simply create a REST API with a Proxy to Lambda. In CDK it is simple like that:

```ts
 const handleUserRequest = new NodejsFunction(this, 'HandleUserRequest', {
 	entry: path.join(`./lambda/handleUserRequest/index.ts`),
 	handler: 'main',
 	runtime: aws_lambda.Runtime.NODEJS_14_X,
 	environment: {
 		QUEUE_TABLE_NAME: queueTable.tableName
 	}
 });

 new apigw.LambdaRestApi(this, 'UserRequestApi', {
 	handler: handleUserRequest
 });
```

#### Lambda for handling user requests

This Lambda handles all user requests. It is directly behind the API Gateway.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649051627673/MhAxTFZAn.png)

The lambda is doing the following steps

```ts
// 1. Deconstruct Query Params

// 2. Build SQS Message (including filtering headers)

// 3. Get Correct Queue

// 4. Send SQS Message

```

For all error or success codes, we can use the `callback` function API Gateway is passing to the lambda function. So, this function passes the message to the appropriate SQS queue. 

#### Execute Final Business Logic

The final lambda function is getting all requests from SQS. Its job is to call the target system (e.g. Vercel Function) and save some metadata.


![Executor Lambda](https://cdn.hashnode.com/res/hashnode/image/upload/v1649051782466/ihFksmeVH.png)

This lambda is doing the following jobs:

```ts
// 1. Mapping over all Records

// 2. Deconstructing the message

// 3. Making the request to the target

// 4. Saving request and response in DynamoDB
```

These are both complete flows. And it shows that the prototype is working very well! With that, I am able to queue a lot of messages, let them run asynchronously and work on the final business logic on my existing system.

## Dashboard

One thing that is completely missing here is the UI. I am building this project together with a friend who bootstrapped some of the initials of the UI. The UI will be implemented with:

- Next.js (React)
- Tailwind CSS
- Hosted on Amplify
- TypeScript

I'll go into more details on the dashboard later.

## Open Points

There are still a lot of open points of course. Some random thoughts here:

1. Is this really a valid problem? -> I do experience this problem a lot so I think it is
2. Is SQS the right service for that? SQS is in nature a polling and not a pushing system. Still, it is the go-to service for message queues.

Technical things I already solved:
-  How many SQS queues can I create per account? -> Indefinite

%[https://twitter.com/sandro_vol/status/1510186383998173189]


-  How many `EventSourceMappings` can I add per account -> Indefinite
- What happens if I redeploy my CDK stack. Do I lose all mappings since they are created via the SDK? -> Won't be overridden, if overridden -> Get ARNs from DDB



## Next Steps

- Building the dashboard
- Building authentication (AppSync + Auth0) 
- Building a dead letter queue
- Give people the opportunity to customize SQS parameter
- Build a landing page
- Look into Vercel integrations

## Final Words

I know there are a lot more things to do. I am not 100% sure if the setup works out like I intend to do it. I am misusing SQS of course a bit since SQS is a polling system and not a pushing system. So it is completely possible that other services are better suited for that. However, SQS is the standard go-to for these use cases so I want to build a better abstraction for that. I do experience this problem a lot myself so I want to try out building this solution and start using it for various use-cases in my daily work. 

Thanks for reading the article. If you want to follow my journey make sure to follow my [Twitter Account](https://twitter.com/sandro_vol)



