## How to Set Up Routes in an Express App: A Step-by-Step Guide

### Introduction

- Setting up routes in an Express application is a fundamental task that helps organize and manage your API endpoints efficiently. In this blog, we'll walk through how to set up and manage routes using router.ts and app.ts files in an industry-standard Express application.

- This is the seventh  blog of my series where i am writing how to write code for industry grade project so that you can manage and scale the project.  

- The first six blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app" and "Creating a Custom Send Response Utility Function in Express". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-not-found-route-in-express-app-1d26

https://dev.to/md_enayeturrahman_2560e3/creating-a-custom-send-response-utility-function-in-express-2fg9

### Organizing Routes in router.ts

The router.ts file is where we consolidate all our module routes, making it easier to manage and scale our application. Create route folder inside app folder and inside that create router.ts File

```javascript
import { Router } from 'express';
import { StudentRoutes } from '../modules/student/student.route';
import { UserRoutes } from '../modules/user/user.route';

const router = Router();

const moduleRoutes = [
  {
    path: '/users',
    route: UserRoutes,
  },
  {
    path: '/students',
    route: StudentRoutes,
  },
];

moduleRoutes.forEach((route) => router.use(route.path, route.route));

export default router;

```

**Explanation:**

- **Import Dependencies:** We import the Router from Express and route modules for students and users.

- **Initialize Router:** We create an instance of the Express Router.

- **Module Routes Array:** An array moduleRoutes holds the paths and respective route handlers. If you want to add further routes in your app then just add the path and route handler in the array. 

- **Register Routes:** We iterate over moduleRoutes and use the router.use method to register each route.

- **Export Router:** Finally, we export the configured router.

### Integrating Routes into the Application in app.ts

The app.ts file is where we set up our Express application, integrate routes, and handle global middleware including error handling.

```javascript
import cors from 'cors';
import express, { Application, Request, Response } from 'express';
import globalErrorHandler from './app/middlewares/globalErrorhandler';
import notFound from './app/middlewares/notFound';
import router from './app/routes';

const app: Application = express();

// Parsers
app.use(express.json());
app.use(cors());

// Application Routes
app.use('/api/v1', router);

// Global Error Handler
app.use(globalErrorHandler);

// Not Found Handler
app.use(notFound);

export default app;
```

**Explanation:**

- **Import Dependencies:** We import necessary packages and modules, including express, cors, our custom middleware, and the consolidated router.

- **Initialize App:** Create an instance of the Express application.

- **Middleware for Parsing:** Set up middleware for parsing JSON and enabling CORS.

- **Application Routes:** Use the app.use method to prefix all routes with /api/v1 and integrate our router.

- **Test Route:** Define a simple test route for the root path.

- **Global Error Handler:** Add a global error handler to catch and handle errors.

- **Not Found Handler:** Add a middleware for handling undefined routes, returning a custom JSON response instead of the default HTML response.

### Conclusion

By following the steps outlined above, you can effectively manage and organize routes in your Express application. This setup ensures that your application remains scalable, maintainable, and easier to debug.