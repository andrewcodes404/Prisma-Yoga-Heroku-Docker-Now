Prisma ~ Yoga ~ Heroku ~ Docker ~ Now ~ Setup & Deploy
======================================================

## to begin...
Really you want to run through these instructions and build from scratch
But if you do clone this you will need to create two .env files
.env 
PRISMA_ENDPOINT=http://localhost:4466
PRISMA_SECRET=#####

.env.prod
PRISMA_ENDPOINT=######
PRISMA_SECRET= 👆the same as above


###Your prisma and yoga are two seperate entities

**Prisma **is a connection to a database that allows you to access the db with graphl framework, it also allows you access to the graphQL playground

**Yoga **is theJS interface that connects prisma with your frontend and allows queris mutations and logic

**In Production**

BOTH need their own server

BOTH need schemas

**Servers**

For prisma and the db we will use Heroku though we could use 

- mongoDB(problems uploading the right schema)

- MySql(you need a dedicated IP, on GGs that is $40/year)

- the prisma Cloud - not for prod) 

**Flow**

Initialise Prisma and create a local DB with docker to use offline

Import Yoga and set that up to use the localDB

Set up our frontend and check we can connect

Set up a DB in Heroku

Create a new instance of Prisma init for production and use this new heroku db

Set scripts to use both dev or prod when we see fit

Upload Yoga to now

Now we wil have a Local development setup and a live production setup

Using Prisma —version 1.34

 👇 👇Set up the mysql local db with docker 👇 👇

<https://www.prisma.io/docs/get-started/01-setting-up-prisma-new-database-JAVASCRIPT-a002/>

you may get an error that the port is already taken, at the moment i’m deleting the other containers

[https://docs.docker.com/engine/reference/commandline/container\_ls/](https://docs.docker.com/engine/reference/commandline/container_ls/)

$ docker container ls

$ docker container kill \<cont ID\>

or prune it. ?

$ docker system prune -f

when we get to

```jsx
$ prisma init --endpoint http://localhost:4466
```

it generates two files prisma.yml and  and datamodel.prisma , the minimal needed to run prisma

```jsx
$ prisma deploy
```

### ☝️that’s it prisma is setup locally and running on

<http://localhost:4466>. And [http://localhost:4466/\_admin](http://localhost:4466/_admin)

Use [http://localhost:4466/\_admin](http://localhost:4466/_admin) to add some users and query them 👇

```jsx
query users {
  users{
    name
  }
}
```

Now lets set up the production version, an online DB with Heroku

[https://app.prisma.io/your-account-name/servers](https://app.prisma.io/andrew-63889b/servers). —\> ADD SERVER

**First Create the DB**

—\> create server —\> create new db —\> heroku —\> update option —\> create database

**Now you have a DB and you need a server for that**

—\> set up server —\> heroku —\> create server

```jsx
$ prisma deploy -n
```

And pick the new DB you created

This will auto change your prisma.yml, comment out the local host

```jsx
#endpoint: http://localhost:4466
endpoint: https://prs-yog-her-now-f085b7bce8.herokuapp.com/PrismaYogaHerokuNow/prod
datamodel: datamodel.prisma
```

We can switch betwen the two by un-commenting out the one we want to use

Lets create some ENV vars and run a script for dev or prod
----------------------------------------------------------

We will need a secret for prod so lets add that to both

create /.env (for dev)

```jsx
PRISMA_ENDPOINT=http://localhost:4466
PRISMA_SECRET=myPrismaSecret9876543210
```

create /env.prod (for prod)

```jsx
PRISMA_ENDPOINT=https://prs-yog-her-now-f085b7bce8.herokuapp.com/PrismaYogaHerokuNow/prod
PRISMA_SECRET=myPrismaSecret9876543210
```

update /prisma.yml

```jsx
endpoint: ${env:PRISMA_ENDPOINT}
secret: ${env:PRISMA_SECRET}
datamodel: datamodel.prisma
```

Now use node to set some scripts

```jsx
npm init -y
```

Update /package.json

```jsx
"scripts": {
    "deploy-dev": "prisma deploy --env-file .env",
    "deploy-prod": "prisma deploy --env-file .env.prod"
  },
```

Now update the data model.prisma

```jsx
type User {
  id: ID! @id
  name: String!
  shoeSize: String!
}
```

and deploy it using the scripts

```jsx
$ npm run deploy-dev
```

your going top get an error

```jsx
Warnings:
  User
    ! You are creating a required field but there are already nodes present
```

You can either force it or delete everything in you db. Deleting everythin might be the better option

☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥☠️ 💀 🔥

You may get an error in the playgournd about a token being out of date

<https://www.prisma.io/forum/t/cannot-open-playground-your-token-is-invalid/2194/4>

This is the answer👇

[https://www.prisma.io/docs/reference/prisma-api/concepts-utee3eiquo\#api-token](https://www.prisma.io/docs/reference/prisma-api/concepts-utee3eiquo#api-token)

you create a token in the CLI $ prisma token

then you manually add it in the [http://localhost:4466/\_admin](http://localhost:4466/_admin). Clik the settings cog

![](resources/88B8407030ED31117E491224E36258A9.jpg)

OK SO FAR WE HAVE….
-------------------

A dev prisma db and server set up locally with docker

A prod prisma db and server set up online with heroku

Prod/Dev env files we can use with a node script to determine wich branch we use

Fixed the TOKEN error in http://db/\_admin

A couple of things to update to get ready for yoga
Update / prisma.yml

```jsx
endpoint: ${env:PRISMA_ENDPOINT}
secret: ${env:PRISMA_SECRET}
datamodel: datamodel.prisma

hooks:
  post-deploy:
      - graphql get-schema -p prisma
```

Create /.graphqlconfig.yml

```jsx
projects:
  app:
    schemaPath: "src/schema.graphql"
    extensions:
      endpoints:
        default: "http://localhost:4444"
  prisma:
    schemaPath: "src/generated/prisma.graphql"
    extensions:
      prisma: prisma.yml
```

Now deploy… this 👆 is going to generate src/generated/prisma.graphql which we need for yoga

```jsx
$ npm run deploy-dev
$ npm run deploy-prod
```

THAT IS ALL FOR PRISMA

NOW YOGA 🧘…
============

some imports

```jsx
npm i dotenv graphql graphql-cli graphql-yoga nodemon prisma prisma-binding
```

### create file —\> src/db.js

Connect directly to Prisma DB with prisma-binding

Allows us to use all the tools in the prisma playground in JS

Typedefs = using the generated API typedef, (this is why we use the hook in .graphqlconfig to pull it done into our project)

```jsx
// This file connects to the remote prisma DB and gives us the ability to query it with JS
const { Prisma } = require('prisma-binding');

// THIS ISN'T IN WESBOS - but you will need it to find the endpoint env
require('dotenv').config();

const db = new Prisma({
  //where are your types/schema
  typeDefs: 'src/generated/prisma.graphql',
  //typeDefs: __dirname + "/schema_prep.graphql",
  // where does the db live
  endpoint: process.env.PRISMA_ENDPOINT,
  secret: process.env.PRISMA_SECRET,
  debug: false,
});

module.exports = db;
```

### create file —\> src/createServer.js

```jsx
const { GraphQLServer } = require("graphql-yoga");

// import the resolvers
const Mutation = require("./resolvers/Mutations");
const Query = require("./resolvers/Queries");

// grab the db
const db = require("./db");
// Create the GraphQL Yoga Server
function createServer() {
  // console.log('createServer() running 🏃‍♂️');
  return new GraphQLServer({
    // ANOTHE SCHEMA 😱 see notes below
    typeDefs: 'src/schema.graphql',
    // typeDefs: __dirname + "/schema.graphql",
    // the above schema will then be matched with an object with the resolvers in
    resolvers: {
      Mutation,
      Query
    },
    // stops some error
    resolverValidationOptions: {
      requireResolversForResolveType: false
    },
    // access the db from the resolvers through 'context'
    context: req => ({ ...req, db })
  });
}

module.exports = createServer;
```

### create file —\> src/schema.graphql  (public facing) OUR API SCHEMA

a schema of mutations/queries that we want to use

```jsx
# import * from './generated/prisma.graphql'
type Mutation {
  createUser(name: String): User!
}

type Query {
  users: [User]!
}
```

SO FAR….

### 

`1. we have created our db`

`2. We have created the server(Prisma) to connect to the db`

`3. Created example user schema, a schema of mutations/queries that we want to us`

```

```

### create file —\> src/index.js

To kick things off

Yoga help [https://github.com/prisma/graphql-yoga\#startoptions-options-callback-options-options--void----null-promisevoid](https://github.com/prisma/graphql-yoga#startoptions-options-callback-options-options--void----null-promisevoid)

What is deets? <https://wesbos.slack.com/threads/convo/C9G96G2UB-1552727085.422700/>

```jsx
//make sure our variables are avaiolable
require('dotenv').config();

//the fn we just created
const createServer = require('./createServer');

//grab the db instance
const db = require('./db');

//run the server fn
const server = createServer();

//using cors here to protect our endpoints
server.start(
  {
    cors: {
      credentials: true,
      origin: process.env.FRONTEND_URL
    },
  },
  // a fn that runs when the server spins
  deets => {
    console.log(`Server is now running on port http://localhost:${deets.port}`);
  }
);
```

### You must create these basic resovers or it won’t run

### create —\> src/resolvers/Mutations.js

```jsx
const Mutations = {
  async createUser(parent, args, ctx, info) {

    const user = await ctx.db.mutation.createUser(
      {
        data: {
          ...args,
        },
      },
      info
    );

    console.log("mutation 🏃‍ args", args);
    return user;
  },
};

module.exports = Mutations;
```

### Create —\> src/resolvers/Queries.js

```jsx
const { forwardTo } = require('prisma-binding');
const Query = {
  users: forwardTo('db'),
};
module.exports = Query;
```

AH HA NEARLY THERE
------------------

### Update /package.json - scripts

```jsx
 "scripts": {
    "start": "nodemon -e js,graphql -x node src/index.js",
    "dev": "nodemon -e js,graphql -x node --inspect src/index.js",
    "deploy-dev": "prisma deploy --env-file .env",
    "deploy-prod": "prisma deploy --env-file .env.prod"
  },
```

$npm run dev

now the yoga wrpapper should be running on localhost:4444

```jsx
query allUsers {
  users{
    name
  }
}
```

PHEW… 😰. What do we have...
----------------------------

### a local dev version of yoga running

db.js. with prisma bindings we connect to the db

createServer.js. we create a server use the db connection and our resolvbers to talk to each other

Schema.graphl We create a schema or queries/mutations that our front end can use

index.js - use this to run things in node

OK NOW THE LETS PUT YOGA ON ZEIT
================================

### We have to re-write the src/generated/prisma.graphql or it wont work on now

In your backend project you have essentially created two apps. one is the prisma server the other is the yoga wrapper

We can use now to deploy our yoga wrapper BUT we have to update the schema so its TS is correct

This is new and from slack Theo Merriamum

<https://wesbos.slack.com/threads/convo/C9G96G2UB-1550829164.025300/>

*"You should not mix graphql imports and js/ts imports. The syntax on the graphql file will be interpreted by graphql-import and will be ignored by ncc (the compiler which reads the \_\_dirname stuff and move the file to the correct directory etc)*

*In my example 'schema\_prep.graphql' is already preprocessed with the imports from the generated graphql file."*

**Create —\> ** src/writeSchema.js:

write the function writeSchema, it copies the schema generated by prism, and places it in the file schema\_prep.graphql

```jsx
const fs = require('fs');
const { importSchema } = require('graphql-import');
const text = importSchema("src/generated/prisma.graphql");
fs.writeFileSync("src/schema_prep.graphql", text)
```

Add a script for writeSchema fn and also add it to the deploy scripts

**Update** —\> package.json

add "&& npm run writeSchema” to the deploy scripts

add ""writeSchema": "node src/writeSchema.js””

```jsx
"scripts": {
    "start": "nodemon -e js,graphql -x node src/index.js",
    "dev": "nodemon -e js,graphql -x node --inspect src/index.js",
    "deploy-dev": "prisma deploy --env-file .env && npm run writeSchema",
    "deploy-prod": "prisma deploy --env-file .env.prod && npm run writeSchema",
    "writeSchema": "node src/writeSchema.js"
  },
```

Now run a script to create the new schema

```jsx
$ npm run writeSchema
```

make sure it ran properly you should have a new file schema\_prep.graphql

 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 
☠️💀💣🗡️ DO THIS OR EVETHING IS F\*\*ked🗡️💣☠️💀
 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 🔥 
------------------------------------------------------------------------------------------------------------------------------------------------------

**UPDAT**E —\> schema.graphql
-----------------------------

**CHANGE **

`# import * from './generated/prisma.graphql’`

**TO THIS**

`# import * from './schema_prep.graphql'`

```jsx
# import * from './schema_prep.graphql'
type Mutation {
  createUser(name: String): User!
}

type Query {
  users: [User]!
}
```

### **Update** —\> src/db.js

With the new schema update the places we use it

```jsx
const db = new Prisma({
    typeDefs: __dirname + "/schema_prep.graphql",
    ...
});
```

### **Update ** —\> src/createServer.js

```jsx
return new GraphQLServer({
        typeDefs: __dirname + '/schema.graphql',
        ...
    });
```

Okay that the schema taken care off, some other stuff you need

### Create /.nowignore

```jsx
node_modules
.vscode
.env
.env.prod
z_notepad.txt
```

### **Create** —\> now.json

```jsx
{
    "version": 2,
    //update this name👇 and delete this comment
    "name": "my-app-name-yoga",
    "builds": [
        { "src": "src/index.js", "use": "@now/node-server" }
    ],
    "routes": [
        { "src": "/.*", "dest": "src/index.js" }
    ],
    "env": {
        "PRISMA_ENDPOINT":"@app_name_prisma_endpoint",
        "PRISMA_SECRET":"@app_name_prisma_secret",
        
        // You dont know this yet as we havent set up the frontend so add it anyway add app_name_frontend_url TBD
        "FRONTEND_URL":"@app_name_frontend_url",
    }
}
```

Add the secrest to now MAKE SURE THEY ARE FROM **.env.prod**
------------------------------------------------------------

**GIVE THEM A UNIQUR HANDLE as they gloabbly sit in your now account. so “fonrtend\_url” is noooo good**

```jsx
$ now secret add myAppName_frontend_url https://myCoolDomainName
```

Deploy to now
-------------

```jsx
$ now
```

