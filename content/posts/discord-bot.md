---
title: "Creating a Discord Bot with Docker"
date: 2020-04-11T19:11:39-07:00
publish-date: 2020-04-11T22:30:00-07:00
draft: false
categories: ["development"]
tags: ["js", "docker"]
---

Ever want to make a Discord Bot and have no idea where to get started? Yeah me too. So I spent most of a night looking into making one and setting it up in a Docker container. Some people thought it was an interesting idea, so here's a write up. This post might get a little long, so I may split it up into a couple of posts. 

## The Basic Idea (A.K.A. What is this and why do I care?)

Discord bots are neat. They do all kinds of things and I've always wondered how they were implemented. So today we're going to build one together. Well, I'm going to build one and explain how along they way. Well, I actually already built it, but I'm building it again to explain it. Well, if you're reading this that means I already finished building it twice, so I guess now it's just you building it. At least you have my commentary to help! Unless my commentary is not useful, in which case, IDK there are other blogs explaining this, so go look at one of those.

Sorry, that was a long winded way of saying nothing. Anyway, here's the plan. We're going to make a bot that will listen on Discord and if someone says "Ping" it will reply with "Pong". We're going to go through the entire process of setting up the bot in Discord, writing the code, turning it into a docker image, pushing the image to Azure Container Registry and then running the docker image on a machine somewhere.

![Hold on to your butts](/images/gifs/hold-on-to-your-butts.gif)


## Getting Started

Before we get started, make sure you have a development environment set up. I'm using [VS Code](https://code.visualstudio.com/) as my IDE, but any text editor will work. You'll also need [nodejs](https://nodejs.org/) (I'm using v12.16.2 right now) and a Discord account. Get one at [discordapp.com](https://discordapp.com/). I'll also be doing this in Typescript since I like actually having a type system, but this can all be done in TS or JS. You will also need [Docker](https://www.docker.com/) if you want to run it in a container like I do in steps 3 & 4 below.

The steps to creating the bot are as follows:
1. [Create a bot on Discord](#1-create-the-bot-on-discord)
2. [Implement the bot code in JS (or TS in our case).](#2-implement-the-bot-code-in-js)
3. [Create the docker image.](#3-create-the-docker-image)
4. [Deploy the docker container and test the bot.](#4-deploy-the-docker-container-and-test-the-bot)

## 1. Create the bot on Discord.

First things first, we need to create a bot in Discord and install it into one of our servers. To create the bot you need to go to the [Discord Developer Portal](https://discordapp.com/developers/applications). To create the bot, we first have to create an "application" in their system. It's a few steps, but it's not too complicated. You'll need to hit the "New Application" button and choose a name for the Application. Just choose whatever name you want for this, I chose "Sample Application" and hit "Create".

You will see a page that looks like this:
![Application general information page](/images/posts/discord-bot/general-information.png)

From this page, copy down the "Client ID" as we'll need that in a minute. Then you can click on the "Bot" link on the left to get to the bot config. You'll need to hit "Add Bot" and confirm it first. This will create the bot for the application.

![Create bot flow](/images/posts/discord-bot/add-bot.png)

Once the bot has been created, you'll need to get the token from the bot. You can do this by clicking on the "Click to Reveal Token" link and copying it, or clicking the "Copy" button. This token is going to be important later, so put it down somewhere safe. **Important: This token should never be shared with anyone as it will give whoever has it full access to the API as the bot user.**

![Create bot flow](/images/posts/discord-bot/bot-details.png)

Once you have the token saved somewhere, we need to add the bot to a server so we can actually test it in the future. We're going to add the bot with Administrator permissions for now, but you can scope the permissions by using the tool at the bottom of the bot page to get the permission number (Administrator is 8). Once you have the number, you'll need to construct a url to add the bot. The URL should look like this:

https://discordapp.com/oauth2/authorize?&client_id=CLIENTID&scope=bot&permissions=PERMISSION

For us the PERMISSION is 8, so the URL will look something like this:

https://discordapp.com/oauth2/authorize?&client_id=0000000000&scope=bot&permissions=8

Copy the URL with your Client ID and paste it into a web browser. From there you can install the bot in a server you are an admin of. You'll know if it worked by logging into Discord and checking if the bot user is a member of the server.

## 2. Implement the bot code in JS 

Now that you have the bot all set up and installed in a server, we can start working on implementing it. Let's create a folder for working on the bot. I called mine `discord-bot`. 

### 2.1 Set up package.json and tsconfig.json

I'm going to launch VS Code in that folder and finish setting up from there. VS Code has a built in terminal that you can open with ``Ctrl+` ``. Let's setup npm with `npm init`. Just enter information in the prompts or leave the defaults. Either is fine for now. This should create a `package.json` file in the file explorer. You can see that the `package.json` contains all the information from the prompts.


![npm init](/images/posts/discord-bot/npm-init.png)

Now we need to install some packages. Run `npm install discord.js --save` to add the Discord JS library to the project. Let's also add `typescript` as a dev dependency with `npm install typescript --save-dev`. You're going to want the Nodejs typings so run `npm install @types/node @types/ws --save-dev` as well. Your `package.json` should now look something like this:

```json
{
  "name": "discord-bot",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "discord.js": "^12.1.1"
  },
  "devDependencies": {
    "@types/node": "^13.11.1",
    "@types/ws": "^7.2.3",
    "typescript": "^3.8.3"
  }
}
```

Next up we need to add a `tsconfig.json` file to tell typescript how to build everything. Run the following command:

```sh
npx tsc --init --rootDir src --outDir dist --target es6 --esModuleInterop --resolveJsonModule --module commonjs --allowJs true --noImplicitAny true
```

This will setup our `tsconfig.json` with some useful defaults. It will look like this:

```json
{
  "compilerOptions": {
    /* Basic Options */
    // "incremental": true,                   /* Enable incremental compilation */
    "target": "es6",                          /* Specify ECMAScript target version: 'ES3' (default), 'ES5', 'ES2015', 'ES2016', 'ES2017', 'ES2018', 'ES2019', 'ES2020', or 'ESNEXT'. */
    "module": "commonjs",                     /* Specify module code generation: 'none', 'commonjs', 'amd', 'system', 'umd', 'es2015', 'es2020', or 'ESNext'. */
    // "lib": [],                             /* Specify library files to be included in the compilation. */
    "allowJs": true,                          /* Allow javascript files to be compiled. */
    // "checkJs": true,                       /* Report errors in .js files. */
    // "jsx": "preserve",                     /* Specify JSX code generation: 'preserve', 'react-native', or 'react'. */
    // "declaration": true,                   /* Generates corresponding '.d.ts' file. */
    // "declarationMap": true,                /* Generates a sourcemap for each corresponding '.d.ts' file. */
    // "sourceMap": true,                     /* Generates corresponding '.map' file. */
    // "outFile": "./",                       /* Concatenate and emit output to single file. */
    "outDir": "dist",                         /* Redirect output structure to the directory. */
    "rootDir": "src",                         /* Specify the root directory of input files. Use to control the output directory structure with --outDir. */
    // "composite": true,                     /* Enable project compilation */
    // "tsBuildInfoFile": "./",               /* Specify file to store incremental compilation information */
    // "removeComments": true,                /* Do not emit comments to output. */
    // "noEmit": true,                        /* Do not emit outputs. */
    // "importHelpers": true,                 /* Import emit helpers from 'tslib'. */
    // "downlevelIteration": true,            /* Provide full support for iterables in 'for-of', spread, and destructuring when targeting 'ES5' or 'ES3'. */
    // "isolatedModules": true,               /* Transpile each file as a separate module (similar to 'ts.transpileModule'). */

    /* Strict Type-Checking Options */
    "strict": true,                           /* Enable all strict type-checking options. */
    "noImplicitAny": true,                    /* Raise error on expressions and declarations with an implied 'any' type. */
    // "strictNullChecks": true,              /* Enable strict null checks. */
    // "strictFunctionTypes": true,           /* Enable strict checking of function types. */
    // "strictBindCallApply": true,           /* Enable strict 'bind', 'call', and 'apply' methods on functions. */
    // "strictPropertyInitialization": true,  /* Enable strict checking of property initialization in classes. */
    // "noImplicitThis": true,                /* Raise error on 'this' expressions with an implied 'any' type. */
    // "alwaysStrict": true,                  /* Parse in strict mode and emit "use strict" for each source file. */

    /* Additional Checks */
    // "noUnusedLocals": true,                /* Report errors on unused locals. */
    // "noUnusedParameters": true,            /* Report errors on unused parameters. */
    // "noImplicitReturns": true,             /* Report error when not all code paths in function return a value. */
    // "noFallthroughCasesInSwitch": true,    /* Report errors for fallthrough cases in switch statement. */

    /* Module Resolution Options */
    // "moduleResolution": "node",            /* Specify module resolution strategy: 'node' (Node.js) or 'classic' (TypeScript pre-1.6). */
    // "baseUrl": "./",                       /* Base directory to resolve non-absolute module names. */
    // "paths": {},                           /* A series of entries which re-map imports to lookup locations relative to the 'baseUrl'. */
    // "rootDirs": [],                        /* List of root folders whose combined content represents the structure of the project at runtime. */
    // "typeRoots": [],                       /* List of folders to include type definitions from. */
    // "types": [],                           /* Type declaration files to be included in compilation. */
    // "allowSyntheticDefaultImports": true,  /* Allow default imports from modules with no default export. This does not affect code emit, just typechecking. */
    "esModuleInterop": true,                  /* Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'. */
    // "preserveSymlinks": true,              /* Do not resolve the real path of symlinks. */
    // "allowUmdGlobalAccess": true,          /* Allow accessing UMD globals from modules. */

    /* Source Map Options */
    // "sourceRoot": "",                      /* Specify the location where debugger should locate TypeScript files instead of source locations. */
    // "mapRoot": "",                         /* Specify the location where debugger should locate map files instead of generated locations. */
    // "inlineSourceMap": true,               /* Emit a single file with source maps instead of having a separate file. */
    // "inlineSources": true,                 /* Emit the source alongside the sourcemaps within a single file; requires '--inlineSourceMap' or '--sourceMap' to be set. */

    /* Experimental Options */
    // "experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
    // "emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */

    /* Advanced Options */
    "resolveJsonModule": true,                /* Include modules imported with '.json' extension */
    "forceConsistentCasingInFileNames": true  /* Disallow inconsistently-cased references to the same file. */
  }
}
```

Now that this is done, we can make a few changes to `package.json` to enable some easier testing and running. Let's update the `"main"` property to `dist/index.js` since we won't be running anything from `/src`. Let's also add some scripts to enable building and running our bot code:
```json
"scripts": {
    "dev": "nodemon --watch 'src/**/*' --exec 'ts-node' src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "debug": "node --inspect dist/index.js"
}
```

Your `package.json` should look like this now:

```json
{
  "name": "discord-bot",
  "version": "1.0.0",
  "description": "",
  "main": "dist/index.js",
  "scripts": {
    "dev": "nodemon --watch 'src/**/*' --exec 'ts-node' src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "debug": "node --inspect dist/index.js"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "discord.js": "^12.1.1"
  },
  "devDependencies": {
    "@types/node": "^13.11.1",
    "typescript": "^3.8.3"
  }
}
```

### 2.2 Adding the bot entry point and connecting to Discord

Now that all we've got typescript and node set up, let's get started on implementation. Let's create a `src` folder and put an `index.ts` file in it. The main part of this file will be creating the Discord client and connecting to Discord with it. Note: Discord.js has a ton of built in functionality and is too much to cover in this blog post. You can read the full docs [here](https://discord.js.org/#/docs/main/stable/general/welcome). I'll cover the code in the file by blocks, so the first thing we want to do is add this:

```ts
import Discord from "discord.js";

// Discord token is required.
if (!process.env.DISCORD_TOKEN) {
    throw new Error("DISCORD_TOKEN environment variable missing.");
}

const onReady = () => {
    // TODO: Implement this
};
const onMessage = (message: Discord.Message) => { 
    // TODO: Implement this
};

const client = new Discord.Client();

client.on('ready', onReady);
client.on('message', onMessage);

const discordToken: string = process.env.DISCORD_TOKEN;

client.login(discordToken);
```

This will bring in the Discord library under the name `Discord`. It also will read in the token from an Environment variable. I don't like hard coding secrets, especially if I'm checking them into source control. So let's throw if the token is not in the environment and set it to a local variable while we do the rest.

The `onReady` and `onMessage` handlers will be invoked once the client starts up and when incoming messages come in. We'll actually implement them a little later.

The `client` is actually the magic that will connect to Discord and calls into our `onReady` and `onMessage` callbacks.

The call to `client.login(discordToken)` will initiate the connection to Discord. At this point, we can add some logging into the `onReady` handler and see what happens when we run it. Update the code with the following:

```ts
const onReady = () => {
    console.log("Connected");
    
    if (client.user) {
        console.log(`Logged in as ${client.user.tag}.`);
    }
}
```

Once that's in, run `npm run build` to generate the `dist` folder with the generated javascript. Once that completes, run `npm run start` to start the bot running locally. You should see something along the lines of this:

```
PS D:\temp-blog\discord-bot> npm run start

> discord-bot@1.0.0 start D:\temp-blog\discord-bot
> node dist/index.js

D:\temp-blog\discord-bot\dist\index.js:9
    throw new Error("DISCORD_TOKEN environment variable missing.");
    ^

Error: DISCORD_TOKEN environment variable missing.
```

This is because we haven't set the environment variable with our token yet. I'm using Powershell, so I'm going to run:
```powershell
$env:DISCORD_TOKEN = 'MyToken'
```

If you run `npm run start` again you should now see the log lines when you bot connects:

```
PS D:\temp-blog\discord-bot> npm run start

> discord-bot@1.0.0 start D:\temp-blog\discord-bot
> node dist/index.js

Connected
Logged in as Sample Application#2809.
```

Go ahead and hit `Ctrl+C` to stop the process.

### 2.3 Handling incoming messages.

Now that we are connected to Discord, we can implement our message handling. As I said earlier, this bot will just respond "Pong" when someone sends "Ping", so let's build a pretty simple handler.

In our `onMessage` method let's add our new code:

```ts
const onMessage = (message: Discord.Message) => { 
    // Don't respond to bots.
    if (message.author.bot) {
        return;
    }

    if (message.content.toLowerCase() == "ping") {
        message.reply("Pong!");
    }
}
```

Now you can run `npm run build` and `npm run start` again. Once you see the "Connected" log show up, go back to your Discord server and send a message "Ping". You should see this response:

![Bot working](/images/posts/discord-bot/bot-output.png)

Now we're ready to start working on the Docker Image.

## 3. Create the Docker Image

The first step to creating the Docker image will be to add a Dockerfile. Lets create a `Dockerfile` at the root of our `discord-bot` folder. Set the content of the Dockerfile to this:

```Dockerfile
FROM node:lts

USER root
ENV APP /usr/src/APP

COPY package.json /tmp/package.json

RUN cd /tmp && npm install --loglevel=warn \
    && mkdir -p $APP \
    && mv /tmp/node_modules $APP

COPY src $APP/src
COPY package.json $APP
COPY tsconfig.json $APP

WORKDIR $APP

RUN npm run build

CMD [ "node", "dist/index.js" ]
```

Let's break down what we're doing in there. We start with setting the base image to `node:lts` to get the current LTS nodejs in our container.

```Dockerfile
USER root
ENV APP /usr/src/APP
```
Here we're setting the user to run as and adding an environment variable with the path we're installing the bot to.

```Dockerfile
COPY package.json /tmp/package.json

RUN cd /tmp && npm install --loglevel=warn \
    && mkdir -p $APP \
    && mv /tmp/node_modules $APP
```

This is copying the `package.json` into a temp folder to run an `npm install` to download our dependencies. Then it copies the dependencies into our `$APP` directory. 

```Dockerfile
COPY src $APP/src
COPY package.json $APP
COPY tsconfig.json $APP

WORKDIR $APP
```

This is copying app files into the final directory and moving our working directory there.

```Dockerfile
RUN npm run build

CMD [ "node", "dist/index.js" ]
```

This builds the bot and then runs it. Now let's actually build our docker image. Go ahead and run this command:

```powershell
docker build . -t discord-bot
```

This will build the image and tag it with `discord-bot`. You can you another tag if you like. Now you can try running your docker image. Remember to pass in your token via the `-e` flag. Here's what my command looks like: 

```powershell
docker run -e DISCORD_TOKEN=$env:DISCORD_TOKEN docker-bot
```

## 4. Deploy the docker container and test the bot.

To deploy the container off of your local machine, you first need to push it to your registry. I'm going to use Azure Container Registry (ACR) since I already have one. You could always publish to DockerHub, but it was easier for me to use my existing registry. If you're interested in using Azure, instructions for setting up a new registry in Azure can be found [here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal). 

Since I'm using ACR for this, I need to tag my image a little differently, so I'm going to run these commands to push it:

```
docker tag discord-bot mysupercoolregistry.azurecr.io/discord-bot
docker push mysupercoolregistry.azurecr.io/discord-bot:latest
```

Now that it's deployed, I can go to my Docker VM host and run the following command to run the image:

```
docker run -e DISCORD_TOKEN='<SecretToken>' --restart unless-stopped mysupercoolregistry.azurecr.io/discord-bot:latest
```

And that's it, we're all set up and running. The bot will always run unless I manually turn it off in Docker. Thanks for reading all the way through. If I feel like it, I'll put up a blog post about how I set up CI/CD to deploy new updates to ACR using Azure DevOps Pipelines and GitHub.