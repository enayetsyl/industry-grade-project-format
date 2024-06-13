## How to handle not found route in express app.


### Introduction

This is the fourth blog of my series where i am writing how to write code for industry grade project so that you can manage and scale the project. In this blog we will learn how to set up a not found in your Express application. 

- The first four blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app" and "Setting up global error handler using next function provided by express". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

- When users attempt to access a route that is not defined in our application, Express sends a default error message. In this tutorial, we will learn how to handle "Not Found" routes using middleware, allowing us to send a custom JSON response instead of the default HTML response provided by Express.

**Step 1: Create the Middleware**
First, create a file named notFound.ts in your middlewares folder and add the following code:

```javascript
import { Request, Response } from 'express';
import httpStatus from 'http-status';

const notFound = (req: Request, res: Response) => {
  return res.status(httpStatus.NOT_FOUND).json({
    success: false,
    message: 'API Not Found !!',
    error: '',
  });
};

export default notFound;
```

- **Explanation:**

  - **Imports:** We import the necessary types from Express (Request, Response) and the http-status library for standard HTTP status codes.

  - **notFound Middleware:** This function handles requests to undefined routes. It takes three parameters:

    - req: The HTTP request object.
    - res: The HTTP response object.
    
  The middleware sends a JSON response with:

    - success: Indicates the failure of the request.
    - message: A custom message indicating that the API endpoint was not found.
    - error: An empty string as there is no error so there is no message.

**Step 2: Integrate the Middleware in app.ts**

Next, in your app.ts file, integrate the newly created middleware:

```javascript
/* eslint-disable no-undef */
/* eslint-disable no-unused-vars */
/* eslint-disable @typescript-eslint/no-unused-vars */
/* eslint-disable @typescript-eslint/no-explicit-any */
import cors from 'cors';
import express, { Application, Request, Response } from 'express';
import globalErrorHandler from './app/middlewares/globalErrorhandler';
import notFound from './app/middlewares/notFound';
import router from './app/routes';

const app: Application = express();

//parsers
app.use(express.json());
app.use(cors());

// application routes
app.use('/api/v1', router);

app.use(globalErrorHandler);

//Not Found
app.use(notFound);

export default app;
```

**Explanation:**

  - Import: Import the notFound middleware from the middlewares folder.

  - Integration: Use the app.use(notFound) method to apply this middleware. This ensures that any request to an undefined route will be handled by the notFound middleware, sending a custom JSON response.


### Conclusion

By following these steps, you can customize the response for undefined routes in your Express application. This approach improves user experience by providing consistent and informative error messages in your API responses.