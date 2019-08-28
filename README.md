# Glip Ping Chatbot

This chatbot is super simple, it replies "pong" whenever it receives "ping".

This chatbot is mainly served as a hello-world style demo bot for developers.

This chatbot is powered by the [ringcentral-chatbot framework for JavaScript](https://github.com/ringcentral/ringcentral-chatbot-js).


## Quick start with AWS Lambda

Please note that, even you are planning to host your bot on AWS Lambda, it is still useful to run an Express.js server during development.
So you might be interested in [Quick start with Express.js](https://github.com/tylerlong/glip-ping-chatbot/tree/express).


### Create an empty project

```
mkdir <bot-project-name>
cd <bot-project-name>
```


### Install dependencies

```
yarn add ringcentral-chatbot pg serverless-http axios
yarn add --dev serverless
```

We installed `pg` because we would like to use PostgreSQL as database.


### Coding

Create `lambda.js` file with the following content:

```js
const createApp = require('ringcentral-chatbot/dist/apps').default
const { createAsyncProxy } = require('ringcentral-chatbot/dist/lambda')
const serverlessHTTP = require('serverless-http')
const axios = require('axios')

const handle = async event => {
  const { type, text, group, bot } = event
  if (type === 'Message4Bot' && text === 'ping') {
    await bot.sendMessage(group.id, { text: 'pong' })
  }
}
const app = createApp(handle)
module.exports.app = serverlessHTTP(app)
module.exports.proxy = createAsyncProxy('app')
module.exports.maintain = async () => axios.put(`${process.env.RINGCENTRAL_CHATBOT_SERVER}/admin/maintain`, undefined, {
  auth: {
    username: process.env.RINGCENTRAL_CHATBOT_ADMIN_USERNAME,
    password: process.env.RINGCENTRAL_CHATBOT_ADMIN_PASSWORD
  }
})

```

You may notice that the code is similar to [its Express.js version](https://github.com/tylerlong/glip-ping-chatbot/tree/express#coding), Especially the `handle` method is identical. This is because we would like to allow code reusing, no matter you deploy your chatbot as Express.js server or to AWS Lambda.


### Create a RingCentral app

Please read the instructions for [how to create a RingCentral chatbot app](https://github.com/tylerlong/ringcentral-chatbot-js#create-a-ringcentral-app).

Leave "OAuth Redirect URI" empty for now because we don't know it until we deploy the project to AWS.


### Specify environment variables:

Create `.env.yml` file using [.lambda.env.yml](https://github.com/tylerlong/ringcentral-chatbot-js/blob/master/.lambda.env.yml) as template.

- `RINGCENTRAL_SERVER` use https://platform.dev.ringcentral.com for sandbox and https://platform.ringcentral.com for production
- `RINGCENTRAL_CHATBOT_CLIENT_ID` & `RINGCENTRAL_CHATBOT_CLIENT_SECRET` could be found in the newly created RingCentral app.
- `RINGCENTRAL_CHATBOT_ADMIN_USERNAME` & `RINGCENTRAL_CHATBOT_ADMIN_PASSWORD` are username and password for administrator.
- `RINGCENTRAL_CHATBOT_DATABASE_USERNAME` & `RINGCENTRAL_CHATBOT_DATABASE_PASSWORD` are username and password for PostgreSQL database.
- `RINGCENTRAL_CHATBOT_DATABASE_CONNECTION_URI` please update "username:password" to correct value according to database credentials above
- Other part should be left untouched


### Create serverless.yml

Create `serverless.yml` with following content:

```yml
service: demo-ping-chatbot

provider:
  stage: prod
  name: aws
  runtime: nodejs10.x
  region: us-east-1
  memorySize: 256
  timeout: 30 # maximum value allowed by api gateway
  environment: ${file(./.env.yml)}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - lambda:InvokeFunction
      Resource: "*"
  vpc:
    securityGroupIds:
      - Ref: LambdaSecurityGroup
    subnetIds:
      - Ref: PrivateSubnet1
      - Ref: PrivateSubnet2
      - Ref: PrivateSubnet3

package:
  exclude:
    - ./**
    - '!lambda.js'
    - '!node_modules/**'
  excludeDevDependencies: true

functions:
  app:
    handler: lambda.app
    timeout: 300 # this value is not allowed by api gateway but we have proxy below to workaround it
    events:
      - http: 'GET /admin/diagnostic'
  proxy:
    handler: lambda.proxy
    events:
      - http: 'ANY {proxy+}'
  maintain:
    handler: lambda.maintain
    events:
      - schedule: rate(1 day)

resources:
  Resources:
    Vpc:
      Type: AWS::EC2::VPC
      Properties:
        CidrBlock: 10.0.0.0/16

    PrivateSubnet1:
      Type: AWS::EC2::Subnet
      Properties:
        AvailabilityZone: us-east-1a
        CidrBlock: 10.0.1.0/24
        VpcId:
          Ref: Vpc
    PrivateSubnet2:
      Type: AWS::EC2::Subnet
      Properties:
        AvailabilityZone: us-east-1b
        CidrBlock: 10.0.2.0/24
        VpcId:
          Ref: Vpc
    PrivateSubnet3:
      Type: AWS::EC2::Subnet
      Properties:
        AvailabilityZone: us-east-1c
        CidrBlock: 10.0.3.0/24
        VpcId:
          Ref: Vpc

    PublicSubnet1:
      Type: AWS::EC2::Subnet
      Properties:
        AvailabilityZone: us-east-1d
        CidrBlock: 10.0.21.0/24
        VpcId:
          Ref: Vpc

    LambdaSecurityGroup:
      Type: "AWS::EC2::SecurityGroup"
      Properties:
        GroupName: ${self:service}-${self:provider.stage}-lambda
        GroupDescription: Allow all outbound traffic, no inbound
        SecurityGroupIngress:
          - IpProtocol: -1
            CidrIp: 127.0.0.1/32
        VpcId:
          Ref: Vpc

    DbSecurityGroup:
      Type: "AWS::EC2::SecurityGroup"
      Properties:
        GroupName: ${self:service}-${self:provider.stage}-db
        GroupDescription: Allow local inbound to port 5432, no outbound
        SecurityGroupIngress:
          - CidrIp: 10.0.0.0/16
            IpProtocol: tcp
            FromPort: 5432
            ToPort: 5432
        SecurityGroupEgress:
          - IpProtocol: -1
            CidrIp: 127.0.0.1/32
        VpcId:
          Ref: Vpc
    DbSubnetGroup:
      Type: "AWS::RDS::DBSubnetGroup"
      Properties:
        DBSubnetGroupName: ${self:service}-${self:provider.stage}
        DBSubnetGroupDescription: Private database subnet group
        SubnetIds:
          - Ref: PrivateSubnet1
          - Ref: PrivateSubnet2
          - Ref: PrivateSubnet3
    Database:
      Type: AWS::RDS::DBInstance
      Properties:
        DBInstanceIdentifier: ${self:service}-${self:provider.stage}
        Engine: postgres
        DBName: dbname
        AllocatedStorage: 20
        EngineVersion: 11.2
        DBInstanceClass: db.t3.micro
        MasterUsername: ${self:provider.environment.RINGCENTRAL_CHATBOT_DATABASE_USERNAME}
        MasterUserPassword: ${self:provider.environment.RINGCENTRAL_CHATBOT_DATABASE_PASSWORD}
        DBSubnetGroupName:
          Ref: DbSubnetGroup
        VPCSecurityGroups:
          - Ref: DbSecurityGroup

    Eip:
      Type: AWS::EC2::EIP
      Properties:
        Domain: vpc
    NatGateway:
      Type: AWS::EC2::NatGateway
      Properties:
        AllocationId:
          Fn::GetAtt:
            - Eip
            - AllocationId
        SubnetId:
          Ref: PublicSubnet1
    PrivateRouteTable:
      Type: AWS::EC2::RouteTable
      Properties:
        VpcId:
          Ref: Vpc
    PrivateRoute:
      Type: AWS::EC2::Route
      Properties:
        RouteTableId:
          Ref: PrivateRouteTable
        DestinationCidrBlock: 0.0.0.0/0
        NatGatewayId:
          Ref: NatGateway
    SubnetRouteTableAssociationPrivate1:
      Type: AWS::EC2::SubnetRouteTableAssociation
      Properties:
        SubnetId:
          Ref: PrivateSubnet1
        RouteTableId:
          Ref: PrivateRouteTable
    SubnetRouteTableAssociationPrivate2:
      Type: AWS::EC2::SubnetRouteTableAssociation
      Properties:
        SubnetId:
          Ref: PrivateSubnet2
        RouteTableId:
          Ref: PrivateRouteTable
    SubnetRouteTableAssociationPrivate3:
      Type: AWS::EC2::SubnetRouteTableAssociation
      Properties:
        SubnetId:
          Ref: PrivateSubnet3
        RouteTableId:
          Ref: PrivateRouteTable

    InternetGateway:
      Type: AWS::EC2::InternetGateway
    VPCGatewayAttachment:
      Type: AWS::EC2::VPCGatewayAttachment
      Properties:
        VpcId:
          Ref: Vpc
        InternetGatewayId:
          Ref: InternetGateway
    PublicRouteTable:
      Type: AWS::EC2::RouteTable
      Properties:
        VpcId:
          Ref: Vpc
    PublicRoute:
      Type: AWS::EC2::Route
      Properties:
        RouteTableId:
          Ref: PublicRouteTable
        DestinationCidrBlock: 0.0.0.0/0
        GatewayId:
          Ref: InternetGateway
    SubnetRouteTableAssociationPublic1:
      Type: AWS::EC2::SubnetRouteTableAssociation
      Properties:
        SubnetId:
          Ref: PublicSubnet1
        RouteTableId:
          Ref: PublicRouteTable
```

### Deploy to AWS

```
npx sls deploy
```


### Update "OAuth Redirect URI" for RingCentral app

Login https://developer.ringcentral.com, navigate to your bot app, open "Settings" tab, set "OAuth Redirect URI" to "https://xxxxxx.execute-api.us-east-1.amazonaws.com/prod/bot/oauth"



### Create database tables

```
curl -X PUT -u admin:password https://<chatbot-server>/prod/admin/setup-database
```

`admin` & `password` are defined in `/env.yml` file we created above.

For more information, please read [setup database](https://github.com/tylerlong/ringcentral-chatbot-js#setup-database).


### [Add the bot to Glip](https://github.com/tylerlong/glip-ping-chatbot/tree/master#add-the-bot-to-glip)


### [Test the bot](https://github.com/tylerlong/glip-ping-chatbot/tree/master#test-the-bot)
