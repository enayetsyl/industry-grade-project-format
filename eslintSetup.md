## How to set up eslint and prettier

The purpose of this blog is to show how to setup eslint in a typescript project. Eslint will help you automatically detect and fix various types of errors in a project. So you development experience will be smooth. 

- Start a project

```javascript
npm init -y
```

- Install necessary packages. Here some packages installed as dev dependency to smooth the development experience. e.g. typescript, nodemon ts-node-dev.

```javascript
npm i express --save
npm i typescript --save-dev
npm i mongoose --save
npm i cors dotenv
npm i -D nodemon
npm i ts-node-dev --save-dev // This will convert ts file to js. 
```

- Folder structure should be as follows initially. There will be another blog that will detail the file and folder structure.

root -> src -> app.ts, server.ts
root -> src -> config -> index.ts
root -> dist
root -> .env, .gitignore, eslint.config.mjs

- Create types and typescript config file

```javascript
tsc --init
npm i --save-dev @types/node
npm i --save-dev @types/express
```

- Modify the typescript config file for the following fields. 

```javascript
// tsconfig.json file
"target": "es2016"
"module": "commonjs"
"rootDir": "./src"
"outDir": "./dist"
```

- Install eslint and type definition 

```javascript
npm i --save-dev eslint @eslint/js typescript typescript-eslint
```

- Update the eslint.config.mjs file as follows

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
- You can browse following link for more rules.
https://eslint.org/docs/latest/rules/

- You can refer eslint doc from the following link
https://eslint.org/docs/latest/

- Install prettier

```javascript
npm install --save-dev prettier
```

- Modify package.json file for script

```javascript
// package.json file
"main": "./dist/server.js"
"scripts":{
  "build": "tsc", // This will build the file that will be helpful for before deploying to production
  "start:prod":"node ./dist/server.js", //For production this script will run the project.
  "start:dev": "ts-node-dev --respawn --transpile-only ./src/server.ts", // For development this script will run the project.
  "lint":"npm eslint .", // It will find out errors in all the files.
  "lint:fix":"npx eslint src --fix", // It will be used for fixing the errors.
     "prettier": "prettier --ignore-path .gitignore --write \"./src/**/*.+(js|ts|json)\"",
    "prettier:fix": "npx prettier --write src",
}
```

- Sample app.ts file will be as follows

```javascript
// app.ts file

import express, { Request, Response } from "express";


const app = express();

//parsers
app.use(express.json());


app.get("/", (req: Request, res: Response) => {
  res.send("Hello from setup file");
});

export default app;
```

- Sample server.ts file will be as follows

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

- Sample index.ts file will be as follows

```javascript
// index.ts file
import dotenv from "dotenv";
dotenv.config();

export default {
  port: process.env.PORT,
  db_url: process.env.DB_URL,
};
```

- Sample .env file will be as follows

```javascript
PORT=5000
DB_URL=your mongodb connection
```

- Sample .gitignore file will be as follows

```javascript
node_modules
.env
```

- You can use following command to run each script

```javascript
npm run build // Do to it before deployment
npm run start:prod // For starting server at production.
npm run start:dev // For starting server at development.
npm run lint // For finding eslint error
npn run lint:fix // For fixing eslint error
npm run prettier // For finding format error
npm run prettier:fix // For fixing prettier error
```