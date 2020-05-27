# AWS Lambdas Simplified
[For more, see official AWS documentation](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)

AWS Lambdas are defined as a function which AWS calls. It's described as serverless architecture, which means devops team doesn't need to take care of any servers - this is responsibility of AWS. So locally there is no [express](https://www.npmjs.com/package/express) server that runs and listens to port via HTTP protocol. There is just a function that needs to be called. 

In node.js, function looks like following (more at https://docs.aws.amazon.com/lambda/latest/dg/lambda-nodejs.html):
```javascript
exports.handler =  async function(event, context) {
  console.log("EVENT: \n" + JSON.stringify(event, null, 2))
  // this is body of my AWS Lamda function. Insert here whatever Lambda needs to do
  return context.logStreamName
}
```
Such function can be triggered via various triggers, but most used are CloudWatch Event Rule when lambda needs to be triggered periodically or API Gateway when lamdba needs to be triggered by HTTP request.

When triggered via API Gateway, lambda function usually wants to provide data to HTTP response. In such case, lambda will look like following:
```javascript
exports.handler =  async function(event, context) {
  console.log("EVENT: \n" + JSON.stringify(event, null, 2))
  // this is body of my AWS Lamda function. Insert here whatever Lambda needs to do
    return new Promise((resolve, reject) => resolve("some data"))
        .then(data => {
            let response = {
                statusCode: 200,
                headers: {
                    Authorization: "some token"
                },
                body: JSON.stringify({
                    myData: data
                })
            }
            return response;
        });
}
```

Event object is whatever trigger puts there. In case of API Gateway, it can be request parameters, path parameters, request headers, etc.