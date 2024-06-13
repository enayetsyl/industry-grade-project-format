
## Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function

### Introduction 

- In any robust Express application, error handling is a critical aspect of maintaining reliability and user experience. Traditionally, writing controller functions involved wrapping asynchronous operations in try-catch blocks to ensure errors were properly caught and handled. However, this approach often led to repetitive boilerplate code across multiple controllers.

- This is the eight blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first seven blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express" and "How to Set Up Routes in an Express App: A Step-by-Step Guide". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-not-found-route-in-express-app-1d26

https://dev.to/md_enayeturrahman_2560e3/creating-a-custom-send-response-utility-function-in-express-2fg9

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-routes-in-an-express-app-a-step-by-step-guide-177j

### Traditional Approach to Writing Controllers

```javascript
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

    // Call to service layer to create student in the database
    const result = await UserServices.createStudentIntoDB(
      password,
      studentData,
    );

    // Send success response
    sendResponse(res, {
      statusCode: httpStatus.OK,
      success: true,
      message: 'Student is created successfully',
      data: result,
    });
  } catch (err) {
    // Pass error to global error handler
    next(err);
  }
};

export const UserControllers = {
  createStudent,
};

```

**Explanation:**

- **createStudent Function:** This function handles the creation of a student entity in a database. It expects parameters req (request), res (response), and next (next middleware function).

- **try-catch Block:** Wraps the asynchronous operation (await UserServices.createStudentIntoDB) to catch any errors that might occur during database interaction.

- **Sending Response:** Upon successful creation, it sends a JSON response using sendResponse utility function with status code 200 (OK), indicating success, a message, and the data returned from the service layer.

- **Error Handling:** If an error occurs during the database operation, it forwards the error (err) to the next middleware function (next(err)), typically the global error handler.

### catchAsync Utility Function

```javascript
import { NextFunction, Request, RequestHandler, Response } from 'express';

const catchAsync = (fn: RequestHandler) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch((err) => next(err));
  };
};

export default catchAsync;

```

**Explanation:**

- **catchAsync Function:** This utility function accepts a request handler function (fn: RequestHandler) as its parameter and returns a new function that handles asynchronous operations.

- **Async Error Handling:** Inside the returned function, it wraps the invocation of fn(req, res, next) in a Promise.resolve() to ensure it always returns a promise.

- **Error Propagation:** If the promise resolves successfully, the response is passed to the next middleware. If it rejects (throws an error), next(err) is called to propagate the error to the global error handler.

- **Middleware:** The above catchAsync.ts file should be kept inside utils folder and it will be used as a middleware by all the controllers in your app. This is the implementation of DRY principle and it will make you code base more cleaner and manageable. 

### Controller Using catchAsync Utility Function

```javascript
import httpStatus from 'http-status';
import catchAsync from '../../utils/catchAsync';
import sendResponse from '../../utils/sendResponse';
import { UserServices } from './user.service';

const createStudent = catchAsync(async (req, res) => {
  const { password, student: studentData } = req.body;

  // Call to service layer to create student in the database
  const result = await UserServices.createStudentIntoDB(password, studentData);

  // Send success response
  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Student is created successfully',
    data: result,
  });
});

export const UserControllers = {
  createStudent,
};

```

**Explanation:**

- **Usage of catchAsync:** Instead of manually wrapping the controller function (createStudent) in a try-catch block, we use catchAsync to handle asynchronous operations and error handling.

- **Simplified Error Handling:** This approach eliminates the need for explicit try-catch blocks in each controller function, reducing boilerplate code and ensuring consistent error handling across the application.

- **Send Response:** Once the database operation completes successfully, it sends a JSON response using sendResponse with status code 200, a success message, and the data returned from the service.

### Benefits of Using catchAsync

- **Code Clarity:** Promotes cleaner and more readable code by abstracting error handling logic into a reusable utility function.

- **Consistent Error Handling:** Ensures that errors are handled uniformly across all controller functions, enhancing maintainability.

- **Enhanced Developer Productivity:** Reduces the amount of repetitive code, allowing developers to focus more on business logic rather than error handling boilerplate.

### Conclusion 

Implementing catchAsync in an Express application streamlines error management and improves code quality, making it a valuable tool for developers building scalable and maintainable APIs. This approach not only simplifies error handling but also improves overall code organization and developer productivity.