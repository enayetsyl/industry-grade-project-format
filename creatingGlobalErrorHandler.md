## Setting up global error handler using next function provided by express

### Introduction

- In this blog i will explain how to set up global error handler in an express app. 

- In Express applications, when an error occurs, it can be passed to the global error handler using the next function. If a parameter is provided to the next function, Express identifies it as an error and forwards it to the global error handler.

- Here's how to set up a global error handler in your Express application:

- **Create the Global Error Handler Middleware: ** First, create a folder named middlewares inside your app folder. Then, create a file named globalErrorHandler.ts with the following content:

```typescript
import { NextFunction, Request, Response } from 'express';

const globalErrorHandler = (
  err: any,
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const statusCode = 500;
  const message = err.message || 'Something went wrong!';

  return res.status(statusCode).json({
    success: false,
    message,
    error: err,
  });
};

export default globalErrorHandler;

```
- This function takes four parameters:
  - err: The error object.
  - req: The request object.
  - res: The response object.
  - next: The next function.

- It sets the status code to 500 by default and uses the error message if provided, otherwise, it defaults to "Something went wrong!". The response is sent in JSON format with properties for success, message, and error.

- **Integrate the Global Error Handler in Your Express App:** In your main application file  app.ts, import and use the global error handler middleware. Ensure it is placed after all other routes and middleware.

```typescript
import cookieParser from 'cookie-parser';
import cors from 'cors';
import express, { Application } from 'express';
import globalErrorHandler from './app/middlewares/globalErrorhandler';
import notFound from './app/middlewares/notFound';
import router from './app/routes';

const app: Application = express();

// Middleware to parse incoming requests
app.use(express.json());
app.use(cookieParser());
app.use(cors({ origin: ['http://localhost:5173'] }));

// Application routes
app.use('/api/v1', router);

// Global error handling middleware
app.use(globalErrorHandler);

//Not Found
app.use(notFound);

export default app;
```

- **Example Usage in a Route:** When an error occurs in a route handler, pass it to next to forward it to the global error handler:

```typescript
import httpStatus from 'http-status';

import { NextFunction, Request, Response } from 'express';
import sendResponse from '../../utils/sendResponse';
import { UserServices } from './user.service';

const createStudent = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  try {
    const { password, student: studentData } = req.body;

const result = await UserServices.createStudentIntoDB(
      password,
      studentData,
    );

    sendResponse(res, {
      statusCode: httpStatus.OK,
      success: true,
      message: 'Student is created succesfully',
      data: result,
    });
  } catch (err) {
    next(err);
  }
};

export const UserControllers = {
  createStudent,
};

```
### Conclusion

- By following these steps, you can set up a global error handler in your Express application, ensuring that all errors are handled consistently and returned to the client in a structured format.