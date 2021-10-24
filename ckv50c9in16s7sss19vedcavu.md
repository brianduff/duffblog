## Cloud Functions with TypeScript

Here's a small (sample that uses TypeScript + Cloud Functions)[https://github.com/brianduff/cloudfunctions].

I have a complicated relationship with JavaScript. I'm old enough to remember when it first appeared in Netscape Navigator back when the web was truly tiny and everything had a kind of boring gray background, and the days when "Dynamic HTML" started to become a thing. It was awful back then. Truly, terribly awful.

At some point in the last few years, I rediscovered JavaScript through the world of Node.js and React, and I can't really imagine using anything other than React to build web apps these days (I tried Vue.js for at least one project, and found it quite lacking).

But the lack of strong typing is bothersome. I successfully strolled along for a while trying to get away with writing React and Node apps without configuring TypeScript, but now I'm all in on TypeScript.

When it comes to the backend, I oscillate a bit between using Rust+Rocket and Node.js with Express. But recently, I've been poking around increasingly with [Google's Cloud Functions](https://developers.google.com/learn/topics/functions). The out of the box instructions though don't do a very good job of explaining how to get things working with TypeScript (there are some [instructions](https://firebase.google.com/docs/functions/typescript), but they're for Firebase, and it's kind of similar, but different enough that it doesn't quite work if you just use Google Cloud directly. 

I got it working, and thought I'd write it down for posterity.

### Setting up the project

First off do the usual npm initialization dance with `npm init -y`.

We're gonna want to install some deps. You'll find the `functions-framework` from Google useful, since it'll let you do local development with Cloud Functions, and also provides types. Additionally as always, we'll want `typescript` itself as a dev dependency:

```
npm install --save-dev @google-cloud/functions-framework typescript
```

Next, let's set up TypeScript. This'll create a `tsconfig.json` file.

```
npx tsc --init
```

I like to make put my code in a `src` folder, and have it output to a separate `ts-built` folder. Here's a minimal `tsconfig.json` file that will achieve that:

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "rootDir": "src",
    "outDir": "ts-built",
  }
}
```

Next, let's write a simple cloud function in `src/index.ts`. Sssh, we're not using any types yet, but as usual with TypeScript, that's ok:

```typescript
// src/index.ts
export const hello = (req, res) => res.send("Hello!")
```

Let's check if it compiles:

```bash
npx tsc
```

If all is well, you should see a `ts-built/index.js` file that contains the compiled JavaScript code.

It'd be cool to see if our Cloud Function is working with the framework. Before we can do that, we need to tell Cloud Functions where to find our `index.js` file. Update `package.json` to set the `main` property to its path:

```json
{
...
   "main": "ts-built/index.js"
...
}
```

Now, we can run the framework:

```
$ npx functions-framework --target=hello
Serving function...
Function: hello
Signature type: http
URL: http://localhost:8080/
```

If you visit the localhost URL, you should be greeted with the expected friendly message.

### Deploying to Google Cloud

When we deploy this to Google Cloud, it will run `npm ci` to build the project. That's just a more efficient way to invoke `npm install`. This isn't enough - we need it to also run the TypeScript compiler, otherwise `index.js` won't exist when the code is up in the Cloud. Luckily, Google Cloud provides a hook to perform extra steps during the build. In `package.json`, we need to add a `gcp-build` script. We'll add a `build` script and a `start` script to make local development easier too:

```json
{
  "scripts": {
    "gcp-build": "npm run build",
    "build": "tsc",
    "start": "npm run build && npx @google-cloud/functions-framework --target=hello"
  }
}
```

Before deploying, it's a good idea to check that it works with `npm run gcp-build`.

To deploy, you'll need to create a Google Cloud project and enable the APIs - the stuff in the "Before you begin" section in (this document)[https://cloud.google.com/functions/docs/quickstart-nodejs#before-you-begin]. You'll also need to install the (Cloud SDK)[https://cloud.google.com/sdk/docs/install] locally so you can use the `gcloud` command. You'll also need to authorize and select your project using (replace PROJECTNAME):

```bash
gcloud auth login
gcloud config set project PROJECTNAME
```

Now you can deploy with:

```bash
gcloud functions deploy simple-function --entry-point hello \
    --allow-unauthenticated --trigger-http --runtime nodejs16
```

It'll churn away for a wee while doing its thing, but after it's done you should be able to go to the console to find out the URL of your new function. In my case, this is https://us-central1-dubh-cloud.cloudfunctions.net/simple-function

### Actually using types!

Ok, so far we've got it working end to end with TypeScript, but we didn't actually use types. You don't actually have to, of course, but if you want to, you can be more specific. For example, the `functions-framework` defines a type called `HttpFunction` for cloud functions themselves, and we could write:

```typescript
import { HttpFunction } from '@google-cloud/functions-framework';

export const hello: HttpFunction = (req, res) => res.send("Hello!")
```

The actual request and response are express types. I'll write an example in a future blog that takes advantage of TypeScript to do much more interesting things. 
