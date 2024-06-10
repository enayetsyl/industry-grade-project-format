### Start project

```javascript
npm init -y
```
```javascript
// package.json file
"main": "./dist/server.js"
"scripts":{
  "start:prod":"node ./dist/server.js",
  "start:dev": "ts-node-dev --respawn --transpile-only ./src/server.ts" 
}
```
```javascript
npm i express --save
npm i typescript --save-dev
npm i mongoose --save
npm i cors dotenv
npm i -D nodemon
npm i ts-node-dev --save-dev
```
```javascript
// .gitignore
node_modules
.env
```
### Folder structure
root -> src-> app.ts, server.ts
root -> src-> config -> index.ts
root -> .env
root -> eslint.config.mjs


### Create ts config file
```javascript
tsc --init
npm i --save-dev @types/node
```
```javascript
// tsconfig.json file
"target": "es2016"
"module": "commonjs"
"rootDir": "./src"
"outDir": "./dist"
```
```javascript
PORT=5000
DB_URL=your mongodb connection
```


```javascript
// app.ts file

import express, { Request, Response } from "express";
import { MovieRoutes } from "./modules/movies/movie.route";
const app = express();

//parsers
app.use(express.json());

app.use("/api/movies", MovieRoutes);

app.get("/", (req: Request, res: Response) => {
  res.send("Hello from setup file");
});

export default app;
```
```javascript
// server.ts file
import mongoose from "mongoose";
import app from "./app";
import config from "./config";

async function main() {
  try {
    await mongoose.connect(config.db_url as string);

    app.listen(config.port, () => {
      console.log(`Example app listening on port ${config.port}`);
    });
  } catch (err) {
    console.log(err);
  }
}

main();
```
```javascript
// index.ts file
import dotenv from "dotenv";
dotenv.config();

export default {
  port: process.env.PORT,
  db_url: process.env.DB_URL,
};
```
### Setting up eslint
```javascript
npm i --save-dev eslint @eslint/js typescript typescript-eslint

```
```javascript
// eslint.config.mjs
import eslint from "@eslint/js"
import tseslint from "@typescript-eslint"

export default tseslint.config(
  eslint.configs.recommended, 
  ...tseslint.configs.recommended,
  {
    languageOptions: {
      globals: {
...globals.node,
      }
    }
  },
  {
    rules: {
      "no-unused-vars": "error",
      "no-undef": "error",
      "prefer-const" : "error",
      "no-console": "warn",
    }
  },
  {
    ignores: ["**/dist/", "**/node_modules/"]
  }
)
```
```javascript
// To run eslint for all files
npx eslint .
// To run eslint for src folder
npx eslint src
// to automate add a script in the package.json file as follow
"lint":"npm eslint ."
// to automating the fixing of the error add following line in the package.json file
"lint:fix":"npx eslint src --fix"
```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript

```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```
```javascript```