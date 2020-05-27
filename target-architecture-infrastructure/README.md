# Target Architecture & Infrastructure

# Outline
- [Components](#components)
- [Infrastructure Decisions](#infrastructure-decisions)
- [Authentication and Authorization](#authentication-and-authorization)
    - [Registration](#registration)
    - [Login](#login)
    - [Authorization](#authorization)
    - [JWT Token basics](#jwt-token-basics)
- [Collecting Data](#collecting-data)
    - [Data Structure - AWS DynamoDB](#data-structure---aws-dynamodb)
    - [Data Source Collector](#data-source-collector)
    - [Data Source Backend](#data-source-backend)


Target architecture - high level
![Target architecture - high level](images/long-term.png "Target architecture - high level")

## Components
* [Frontend](https://github.com/lukasorgovan/metrixs-fe) - UI of metrics application
* [Middleware / Aggregator](https://github.com/msolanikova/matrix-app-aggregator) - responsible for aggregating calls to all backends and providing aggregated data in single response to frontend
* [Data source]  backend - responsible for providing data from DB (possibly doing some calculations / data aggregations)
* [Data source] collectors - responsible for collecting data from clients' data sources and storing it in DB
* auth - for authenication and authorization

## Infrastructure Decisions
In order for system to require as little maintenance as possible, its infrastructure needs to be simple to understand even for aws beginners. Also it needs to be able to automatically scale. One option is to provide all applications in form of docker images, but this approach would require adopting service like [AWS ECS](https://aws.amazon.com/ecs/) or [AWS EKS](https://aws.amazon.com/eks/) which are not very easy to configure (load balancers, scalability, health checks, ...)

So till performance will be acceptable, services will be deployed in form of [AWS Lamdas](https://aws.amazon.com/lambda/)

## Authentication and Authorization
For user management AWS Cognito will be used as it is cheap, easy to use and secure auth service. It supports OAuth flows and built-in UI for login (+ a lot of other great features). 

New user pool must be created and configured. Important configuration:
* General settings -> Attributes: name attribute should be selected
* General settings -> Attributes: custom attribute `clientId` must be added (non mutable as user should not be able to change which client he belongs to)
* General settings -> Policies: "Only allow administrators to create users"
* General settings -> App clients: new app client must be created; preferably with no secret key as some flows do not support secret keys
* App integration -> App client settings: "Cognito user pool" should be selected
* App integration -> App client settings: "Refresh token expiration (days)" should be set to some valid value, e.g. 1 day. So it will be possible to refresh (re-generate) expired token just for one day. After that user will have to login again. 
* App integration -> App client settings: "Callback URL" should be set to UI of Metrics App
* App integration -> App client settings: "Authorization code grant" should be selected
* App integration -> Domain name: name must be set

### Registration
Registration is currently an open point (:question:) as it should include payments. Till this is resolved, registration can be a manual process of adding new user to cognito.

Adding new user via AWS CLI:
```
aws cognito-idp admin-create-user --user-pool-id {pool id} --username {username/email to add} --user-attributes Name=custom:clientId,Value={client id} --profile {aws cli profile with cognito permissions}
```
* pool id - in General settings
* username / email to add - email of user being added
* client id - id of client this user belongs to. Value should be from `clients` DynamoDB table

### Login
AWS Cognito provides fully working OAuth 2.0 service and also provides built-in Login UI. Style of this UI is customizable, so can be branded to Metrics App. It also contains and fully supports "Forgot password" flow and "Change password of firts login" flow. If built-in login UI wouldn't be used, these flows would have to be implemented manually using cognito APIs. 

**Metrics App UI**:
* must send JWT token (id token) issued by AWS Cognito with all requests to backend. This is then validated on API Gateway part
* must request new token once current one has expired. For this, refresh token issued by Cognito must be used. Check ["Exchanging a Refresh Token for Tokens"](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html)
* must redirect to login once it's not possible to refresh expired token

Schematic login using AWS Cognito
![Login](images/login.png "Login")

**API Gateway for backend APIs**:
* must have Authorizer configured pointing to given AWS Cognito pool (Authorization header should be used to pass a token to API Gateway)
* all backend resources must be configured to use given Authorizer (API Gateway -> Resources -> method of any endpoint -> Method Request -> Authorization; or via AWS SAM, see [secured backend lambda function template in examples](https://github.com/msolanikova/matrix-app-aggregator-lambda-example/blob/master/template.yaml#L82))
* must base64 decode Authorization header value to get cliend id of logged in user (see [secured backend lambda function in examples](https://github.com/msolanikova/matrix-app-aggregator-lambda-example/blob/master/metrics-secured-backend/secured-backend.js#L38))

Requesting data
![Requesting data](images/requesting-data.png "Requesting data")

Requesting data with expired or invalid id token
![Requesting data with expired or invalid id token](images/invalid-token.png "Requesting data with expired or invalid id token")

### Authorization
For simple functionality at the beginning there is no need to have any kind of authorization. However if there is a requirement with authorization needs, e.g. to allow clients to create their own users, it could be achieved using following approach. 

* AWS Cognito supports adding users to predefined groups. So there could be `admins` group for users with permissions to create users and `users` group without these permissions. 
* or custom attribute `role` can be added to AWS Cognito attributes

Functionality:
* new AWS Lambda function should be created, connected to API Gateway secured using Authorizer for given Cognito pool
* at the beginning of its execution this lambda must check which group (or role) user has and would allow to continue only to admins
* this function would then create users based on request body to client there logged in user belongs (and it would be possible to create only users in group `users` or with `users` role)
* there could be UI created for that and allowed only to users in `admins` group or with `admin` role
  * Metrics App UI is able to recognize group or role of given user by decoding id token 

### JWT Token basics
Contains 3 parts:
- header (algorithm and token type) - always the same
```
{
  "alg": "HS512",
  "typ": "JWT"
}
```
- payload - data that Auth service want to share with Frontend and other services; should contain client id, username, user's name, user's role (authorization :question:), iat - issued at for possible expiration
```
{
  "custom:clientId": "21fdsfsdf323-fdfsdds3-sdfsdf3",
  "username": "john.doe@domain.com",
  "cognito:groups": ["admins"],
  "name": "John Doe",
  "iat": 1516239022,
  ...
}
```
- signature - hash with algorithm from header of header and payload with some secret
```
HMACSHA512(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
your-512-bit-secret
)
```
Authorization header then contains token:
`base64UrlEncode(header).base64UrlEncode(payload).signature`

Example:
`eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJjdXN0b206Y2xpZW50SWQiOiIyMWZkc2ZzZGYzMjMtZmRmc2RkczMtc2Rmc2RmMyIsInVzZXJuYW1lIjoiam9obi5kb2VAZG9tYWluLmNvbSIsImNvZ25pdG86Z3JvdXBzIjpbImFkbWlucyJdLCJuYW1lIjoiSm9obiBEb2UiLCJpYXQiOjE1MTYyMzkwMjJ9.v-5umQxLTvOKpHTBxiGBJUxkyygSaOmAQRX0JZIc50iF26RRc9a996mATHtMq9qTxMi3030URqcgNEeNQ3lGbQ`


## Collecting data
Collectors are responsible for periodically connecting to given client's data source and storing retrieved data in DB. Data doesn't have to be stored in raw form which data source API provides, the form can be adjusted to fit metrics application. E.g. jira collector doesn't need to store each single issue, its worklog, etc. It can store just worklogs of team members. 

Each client **MUST** have its own database. Everything can run on single database server, but clients data needs to be separated. 

clients DB holds data and configuration of all clients and their users while client-[client id] database holds data from all data sources of given client.

### Data Structure - AWS DynamoDB
- **clients**
  - clients’ settings
  - ready in AWS
  - primary key:
    - partition key: clientId 
    - no sort key
  - other properties (same as in settings.json currently used)

- **{clientId}-teams** – may or may not be useful
  - ready in AWS
  - primary key:
    - partition key: teamId
    - no sort key

- **{clientId}-commits**
  - primary key:
    - partition key: username – github username of commit author
    - sort key: timestamp – unix timestamp of commit (number)
  - repository
  - sha (for debug purposes)
  - stats (map)
    - additions (number)
    - changedFiles (number)
    - deletions (number)
  - teamId
  - prNumber – number of pull request if given commit was part of PR
  - secondary index by teamId and timestamp
  - example: client1-commits
  - for endpoints:
    - /github/user-commits
    - /github/team-commits
    - /github/user-impact
    - /github/team-impact
    - /github/team-scores
    - /github/commits-without-requests

- **{clientId}-activedays**
  - primary key:
    - partition key: username – github username of commit author
    - sort key: day – string representation of day when commit was created or number as timestamp of day when commit was created (with time 00:00:00.000)
  - teamId
  - secondary index by teamId and day
  - example: client1-activedays
  - for endpoints:
    - /github/user-active-days
    - /github/team-active-days
    
- **{clientId}-comments**
  - primary key:
      - partition key: commentId
      - sort key: timestamp - when comment was created
  - username – github username of commit author
  - prNumber
  - teamId
  - secondary index by username and timestmap
  - for endpoints:
    - /github/user-comments-count

- **{clientId}-sprints**
  - primary key:
    - partition key: sprintId – jira id
    - sort key: start – unix timestamp
  - end
  - sprintName
  - commitment
  - velocity
  - trend
  - teamId
  - secondary index by teamId and start
  - example: client1-sprints
  - for endpoints:
    - /jira/velocity
    - /jira/sprints

- **{clientId}-issues**
  - primary key:
    - partition key: sprintId – jira id
    - sort key: issuesId – jira id
  - aggregateTimeSpent
  - aggregateTimeOriginalEstimate
  - issueType
  - resolutionDate
  - secondary index by teamId and resolutionDate
  - example: client1-issues
  - for endpoints:
    - /jira/estimations
    - /jira/throughput
    - /jira/quality

- **{clientId}-worklogs**
  - primary key:
    - partition key: worklogId – jira id
    - sort key: started – unix timestamp
  - accountId
  - issueId
  - sprintId
  - teamId
  - timeSpent
  - secondary index by teamId and started
  - example: client1-worklogs
  - for endpoints:
    - /jira/focus


### [Data source] collector
- for now metrics application supports just jira and github, but it future it might support also sonar, jenkins or other data sources

Each client should have its own collectors (single collector connects to just single customer's data source)

Data source collector should be deployed in form of [AWS Lambda](https://aws.amazon.com/lambda/) function, but **WATCH OUT**, there are limits of how long labda function can run. 

The trigger for data source collector lambda function should be CloudWatch Event Rule - cron expression.

Example of data collectors:
* jira issue collector for client 1
* github commit collector for client 1
* jira issue collector for client 2
* jira worklog collector for client 3
* github commit collector for client 2

Same collectors for different clients can have exactly same cron expressions in event rules (triggered at the same time).

**Changes of the past data**

Past data in the data source can be changed, e.g. when pull request is merged to master and commits in the past are now part of master at the point of pull request merge or when worklog is added for the past period. 

In order to have current data, data in the past needs to be loaded periodically multiple times as it's not possible to query data source for just changed data. It's been decided that past 28 days will be loaded (usual length of 2 sprints). Everyday data for past 28 days will be loaded. It means data for day 0 will be loaded during day 1, during day 2, ..., during day 28.

 Data will be loaded DAILY (once a day during the night) until specified by client otherwise.
 ![Storing data](images/storing-data.png "Storing data")

### [Data source] backend
REST API providing data from DB to middleware. 

Each data source has its own backend (jira backend, github backend, etc.) which loads data from DB based on client information provided in JWT in Authorization header.

Each endpoint is deployed as AWS Lambda function and is connected to single API Gateway endpoint.