## How to Handle Errors in an Industry-Grade Node.js Application

- This is the thirteenth blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first twelve blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide", "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function", "Understanding Populating Referencing Fields in Mongoose", "Creating a Custom Error Class in an express app", "Understanding Transactions and Rollbacks in MongoDB" and "Updating Non-Primitive Data Dynamically in Mongoose". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-not-found-route-in-express-app-1d26

https://dev.to/md_enayeturrahman_2560e3/creating-a-custom-send-response-utility-function-in-express-2fg9

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-routes-in-an-express-app-a-step-by-step-guide-177j

https://dev.to/md_enayeturrahman_2560e3/simplifying-error-handling-in-express-controllers-introducing-catchasync-utility-function-2f3l

https://dev.to/md_enayeturrahman_2560e3/understanding-populating-referencing-fields-in-mongoose-jhg

https://dev.to/md_enayeturrahman_2560e3/creating-a-custom-error-class-in-an-express-app-515a

https://dev.to/md_enayeturrahman_2560e3/understanding-transactions-and-rollbacks-in-mongodb-2on6

https://dev.to/md_enayeturrahman_2560e3/updating-non-primitive-data-dynamically-in-mongoose-17h2


Proper error handling is crucial for developing a robust, industry-grade application. In order to handle errors effectively, we first need to understand the different types of errors that can occur. Errors can be categorized as follows:

**Operational Errors:** These are errors that can be anticipated during normal operations:

  - Invalid user inputs.
  - Failed server startup.
  - Failed database connection.
  - Invalid authentication token.

**Programmatical Errors:** These are errors introduced by developers during development:

  - Using undefined variables.
  - Accessing non-existent properties.
  - Passing incorrect types to functions.
  - Using req.params instead of req.query.

**Unhandled Rejections:** These occur when promises are rejected and not handled.

**Uncaught Exceptions:** These occur when errors in synchronous code are not caught.

Operational and programmatical errors can be managed within an Express app using a global error handler, throwing new errors, or using the next function. However, unhandled rejections and uncaught exceptions can occur inside or outside of an Express application, requiring proper handling.

Errors in an application are not file-specific. They can originate from routes, controllers, services, validations, utilities, or other files.

Each type of error follows a unique pattern. For example, Zod provides error details in an object with an issues array, while Mongoose provides error details in an errors object. Additionally, Mongoose cast errors and duplicate errors have distinct patterns.

Sending raw errors directly to the frontend makes it difficult to handle them uniformly. Therefore, we need to format all possible errors at the backend and send a common pattern to the frontend. This involves sending all errors to a global error handler, creating specific handlers for each type of error, converting them to a common pattern, and then sending a response to the user. The frontend will receive a consistent error structure: success, message, errorSources, and stack. The stack field, which pinpoints the error, will only be sent in the development environment to avoid increasing the application's vulnerability in production.

### Creating a Global Error Handler

First, we will create a globalErrorHandler.ts file:

```javascript
/* eslint-disable @typescript-eslint/no-unused-vars */
/* eslint-disable no-unused-vars */
import { ErrorRequestHandler } from 'express';
import { ZodError } from 'zod';
import config from '../config';
import AppError from '../errors/AppError';
import handleCastError from '../errors/handleCastError';
import handleDuplicateError from '../errors/handleDuplicateError';
import handleValidationError from '../errors/handleValidationError';
import handleZodError from '../errors/handleZodError';
import { TErrorSources } from '../interface/error';

const globalErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  // Setting default values
  let statusCode = 500;  // Default status code if none is received
  let message = 'Something went wrong!';  // Default message if none is received
  let errorSources: TErrorSources = [
    {
      path: '',
      message: 'Something went wrong',
    },
  ];  // Default error sources if none is received

  if (err instanceof ZodError) { // Check if error is an instance of ZodError
    const simplifiedError = handleZodError(err); // Format error using handleZodError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
  } else if (err?.name === 'ValidationError') { // Check if error is a Mongoose validation error
    const simplifiedError = handleValidationError(err); // Format error using handleValidationError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
  } else if (err?.name === 'CastError') { // Check if error is a Mongoose cast error
    const simplifiedError = handleCastError(err); // Format error using handleCastError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
  } else if (err?.code === 11000) { // Check if error is a Mongoose duplicate key error
    const simplifiedError = handleDuplicateError(err); // Format error using handleDuplicateError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
  } else if (err instanceof AppError) { // Check if error is an instance of AppError
    statusCode = err?.statusCode; // Set status code from error
    message = err.message; // Set message from error
    errorSources = [
      {
        path: '',
        message: err?.message,
      },
    ];
  } else if (err instanceof Error) { // Check if error is a generic error
    message = err.message; // Set message from error
    errorSources = [
      {
        path: '',
        message: err?.message,
      },
    ];
  }

  // Return response
  return res.status(statusCode).json({
    success: false,
    message,
    errorSources,
    err,
    stack: config.NODE_ENV === 'development' ? err?.stack : null,
  });
};

export default globalErrorHandler;

```
### Handling Zod Errors

```javascript
// globalErrorhandler.ts file
import { ZodError } from 'zod'; // Importing ZodError from zod.
import handleZodError from '../errors/handleZodError'; // Importing handleZodError file

if (err instanceof ZodError) { // Check if error is an instance of ZodError
    const simplifiedError = handleZodError(err); // Format error using handleZodError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
}

// handleZodError.ts file
import { ZodError, ZodIssue } from 'zod'; // Importing ZodError and ZodIssue type from zod. ZodIssue is the type for each item inside the issues array given by zod error.  
import { TErrorSources, TGenericErrorResponse } from '../interface/error'; // Importing type declared for error sources and generic error response.

const handleZodError = (err: ZodError): TGenericErrorResponse => {
  const errorSources: TErrorSources = err.issues.map((issue: ZodIssue) => { // Loop through issues array provided by Zod
    return {
      path: issue?.path[issue.path.length - 1], // Zod provides the path at the last index of path array. Retrieve it.
      message: issue.message, // Retrieve message from issue.
    };
  });

  const statusCode = 400; // Set the status code

  return { // Return status code, fixed message, and errorSources
    statusCode,
    message: 'Validation Error',
    errorSources,
  };
};

export default handleZodError;

```

### Handling Mongoose Validation Errors

```javascript
// globalErrorhandler.ts file
else if (err?.name === 'ValidationError') { // Check if error is a Mongoose validation error
    const simplifiedError = handleValidationError(err); // Format error using handleValidationError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
}

// handleValidationError.ts file
import mongoose from 'mongoose';
import { TErrorSources, TGenericErrorResponse } from '../interface/error';

const handleValidationError = (
  err: mongoose.Error.ValidationError, // Importing type from Error property within mongoose
): TGenericErrorResponse => { // TGenericErrorResponse is a type for the return so that same style is followed by the different error handler and maintain consistency. 
  const errorSources: TErrorSources = Object.values(err.errors).map( // Inside the err object there is a property named errors and its value is an array i mapped its value here
    (val: mongoose.Error.ValidatorError | mongoose.Error.CastError) => {
      return {
        path: val?.path, // Extract path from the val object
        message: val?.message, // Extract message from the val object
      };
    },
  );

  const statusCode = 400; // Set the status code

  return { // Return status code, fixed message, and errorSources
    statusCode,
    message: 'Validation Error',
    errorSources,
  };
};

export default handleValidationError;

```
### Handling Mongoose Cast Errors

```javascript
// globalErrorhandler.ts file
else if (err?.name === 'CastError') { // Check if error is a Mongoose cast error
    const simplifiedError = handleCastError(err); // Format error using handleCastError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
}

// handleCastError.ts file
import mongoose from 'mongoose';
import { TErrorSources, TGenericErrorResponse } from '../interface/error';

const handleCastError = (
  err: mongoose.Error.CastError, // Importing type from Error property within mongoose
): TGenericErrorResponse => {
  const errorSources: TErrorSources = [
    {
      path: err.path, // Extract path from the err object
      message: err.message, // Extract message from the err object
    },
  ];

  const statusCode = 400; // Set the status code

  return { // Return status code, fixed message, and errorSources
    statusCode,
    message: 'Invalid ID',
    errorSources,
  };
};

export default handleCastError;

```
### Handling Mongoose Duplicate Key Errors

```javascript
// globalErrorhandler.ts file
else if (err?.code === 11000) { // Check if error is a Mongoose duplicate key error
    const simplifiedError = handleDuplicateError(err); // Format error using handleDuplicateError
    statusCode = simplifiedError?.statusCode; // Set status code from formatted error
    message = simplifiedError?.message;  // Set message from formatted error
    errorSources = simplifiedError?.errorSources; // Set error sources from formatted error
}

// handleDuplicateError.ts file
import { TErrorSources, TGenericErrorResponse } from '../interface/error';

const handleDuplicateError = (err: any): TGenericErrorResponse => {
  // Extract the duplicated field name from the error message
  const match = err.message.match(/"([^"]*)"/); // Use regex to extract field name within quotes
  const extractedMessage = match && match[1]; // Extract the first matched group from the regex match

  const errorSources: TErrorSources = [
    {
      path: '', // No specific path for duplicate error
      message: `${extractedMessage} already exists`, // Customize the error message
    },
  ];

  const statusCode = 400; // Set the status code

  return { // Return status code, fixed message, and errorSources
    statusCode,
    message: 'Duplicate Key Error',
    errorSources,
  };
};

export default handleDuplicateError;

```
### Handling Unhandled Rejections and Uncaught Exceptions

Unhandled rejections and uncaught exceptions can cause your application to crash if not properly handled. Here's how to handle them:

```javascript
import { Server } from 'http';
import mongoose from 'mongoose';
import app from './app';
import config from './config';

let server: Server;

async function main() {
  try {
    await mongoose.connect(config.database_url as string); // Connect to the database

    server = app.listen(config.port, () => { // Start the server
      console.log(`App is listening on port ${config.port}`);
    });
  } catch (err) {
    console.log(err); // Log the error if database connection fails
  }
}

main();

// Handle unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  console.log('Unhandled Rejection at:', promise, 'reason:', reason);
  if (server) {
    server.close(() => {
      process.exit(1); // Exit the process after server closes
    });
  } else {
    process.exit(1); // Exit the process immediately if server is not running
  }
});

// Handle uncaught exceptions
process.on('uncaughtException', (err) => {
  console.log('Uncaught Exception thrown:', err);
  process.exit(1); // Exit the process immediately
});

```
### Conclusion

By understanding and properly handling different types of errors, we can create a more robust and maintainable Node.js application. Each type of error, whether from Zod, Mongoose, or other sources, requires a specific handling strategy to ensure consistency and clarity in error responses. By implementing a global error handler and individual error handlers, we can ensure that our application provides informative and consistent error messages to the frontend, improving both user experience and developer productivity.





// Below is the draft writting


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




- Proper error handling is crucial for developing an industry-grade app. In order to handle errors properly we have to know them first. Errors can be categorized as follows:

  - Operational error: Errors that can predict will happen in the future:
    - Invalid user inputs.
    - Failed to run server.
    - Failed to connect to database.
    - Invalid auth token

  - Programmatical error: Errors that developers produce when developing app.
    - Using undefined variables.
    - Using properties that do not exist.
    - Passing number instead of string.
    - Using req.params instead of req.query.
  
  - Unhandled Rejection: For asynchronous code when rejection is not dealt with. 

  - Uncaught Exception: When error occurred in the synchronous code. 

The operational and programmatical error can be deal by express app using global error handler or throw new error or using next function. but Unhandled rejection and uncaught exception can occur inside/outside of express application we have to deal them properly.

Error in an application is not file specific. It can come from route, controller, service, validation, utilities or any other files.

Each error has its own pattern. zod provide error details in an object which has a filed name issues that's value is an array. on the other hand mongoose provide error details in an object that has errors property with an object as value, inside that there is a name property that has an object as a value and it contains the error details. again mongoose cast error and duplicate error has completely different pattern.

- if we send each error as it is to the frontend then it will be difficult to handle at the frontend. so we have to format all the possible errors at the backend and send a common pattern to the frontend.

In order to do that first we will send all the errors to the global error handler then over there we will create different handler to handle each type of error and convert them to a common pattern and then send response to the user. so front end will receive same pattern for all types of error which is success, message, errorSources and stack. stack pin point the error so we will send it only in development environment. we will not send it at production environment as it will increase the vulnerability of our app.

- We will create globalErrorhandler.ts file as follows

```javascript
/* eslint-disable @typescript-eslint/no-unused-vars */
/* eslint-disable no-unused-vars */
import { ErrorRequestHandler } from 'express';
import { ZodError } from 'zod';
import config from '../config';
import AppError from '../errors/AppError';
import handleCastError from '../errors/handleCastError';
import handleDuplicateError from '../errors/handleDuplicateError';
import handleValidationError from '../errors/handleValidationError';
import handleZodError from '../errors/handleZodError';
import { TErrorSources } from '../interface/error';

const globalErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  //setting default values
  let statusCode = 500;  // Set default status code if no status code is received
  let message = 'Something went wrong!';  // Set default message if no message is received
  let errorSources: TErrorSources = [
    {
      path: '',
      message: 'Something went wrong',
    },
  ];  // Set default error sources if no error sources is received. its type is declared i the interface folder in error.ts file

// I will explain each of the following if, if else and else block later together with their corresponding handler file.
  if (err instanceof ZodError) { //As zod provides error as a sub class of ZodError, so we are checking that whether err has instanceof ZodError.
    const simplifiedError = handleZodError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSources = simplifiedError?.errorSources;
  } else if (err?.name === 'ValidationError') {
    const simplifiedError = handleValidationError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSources = simplifiedError?.errorSources;
  } else if (err?.name === 'CastError') {
    const simplifiedError = handleCastError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSources = simplifiedError?.errorSources;
  } else if (err?.code === 11000) {
    const simplifiedError = handleDuplicateError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSources = simplifiedError?.errorSources;
  } else if (err instanceof AppError) {
    statusCode = err?.statusCode;
    message = err.message;
    errorSources = [
      {
        path: '',
        message: err?.message,
      },
    ];
  } else if (err instanceof Error) {
    message = err.message;
    errorSources = [
      {
        path: '',
        message: err?.message,
      },
    ];
  }

  //ultimate return
  return res.status(statusCode).json({
    success: false,
    message,
    errorSources,
    err,
    stack: config.NODE_ENV === 'development' ? err?.stack : null,
  });
};

export default globalErrorHandler;

```

- zod error in globalErrorhandler.ts and handleZodError.ts file

```javascript
// globalErrorhandler.ts file
import { ZodError } from 'zod'; // Importing ZodError from zod.
import handleZodError from '../errors/handleZodError'; // Importing handleZodError file

if (err instanceof ZodError) { //As zod provides error as a sub class of ZodError, so we are checking that whether err has instanceof ZodError.
    const simplifiedError = handleZodError(err); // Passing the err to handleZodError file for formating
    statusCode = simplifiedError?.statusCode; // setting the status code from simplifiedError
    message = simplifiedError?.message;  // setting the message from simplifiedError
    errorSources = simplifiedError?.errorSources; // setting the error sources from simplifiedError
  } 

  // handleZodError.ts file

  import { ZodError, ZodIssue } from 'zod'; // Importing ZodError and ZodIssue type from zod. ZodIssue is the type for each item inside the issues array given by zod error.  
import { TErrorSources, TGenericErrorResponse } from '../interface/error'; // Importing type declared for error sources and generic error response.

const handleZodError = (err: ZodError): TGenericErrorResponse => {
  const errorSources: TErrorSources = err.issues.map((issue: ZodIssue) => { // from the issues array provided by zod  i looped each issue and stored in errorSources variable. 
    return {
      path: issue?.path[issue.path.length - 1], //Zod always provide the path at the last index of path array. Here we are retriving it.
      message: issue.message, //Retriving message from issue. 
    };
  });

  const statusCode = 400;// Setting the status code

  return { // returning status code, fixed message and errorSources
    statusCode,
    message: 'Validation Error',
    errorSources,
  };
};

export default handleZodError;
```

- Below i will explain the if block for mongoose validation error in the globalErrorhandler.ts file and handleValidationError.ts file

```javascript
// globalErrorhandler.ts file

else if (err?.name === 'ValidationError') {// mongoose name the error as ValidationError in the name property so in the else if block i checked it. 
    const simplifiedError = handleValidationError(err); //passed the error to the handleValidationError function to the handleValidationError.ts file
    statusCode = simplifiedError?.statusCode; // setting the status code from simplifiedError
    message = simplifiedError?.message;  // setting the message from simplifiedError
    errorSources = simplifiedError?.errorSources; // setting the error sources from simplifiedError
  }

// handleValidationError.ts file

import mongoose from 'mongoose';
import { TErrorSources, TGenericErrorResponse } from '../interface/error';

const handleValidationError = (
  err: mongoose.Error.ValidationError, // Importing type from Error property within mongoose
): TGenericErrorResponse => { // TGenericErrorResponse is a type for the return so that same style is followed by the different error handler and maintain consistency. 
  const errorSources: TErrorSources = Object.values(err.errors).map( // Inside the err object there is a property named errors and its value is an array i mapped its value here
    (val: mongoose.Error.ValidatorError | mongoose.Error.CastError) => {
      return {
        path: val?.path, // Extarction path from the val object
        message: val?.message, // Extarction message from the val object
      };
    },
  );

// Below is the same as zod validation error.
  const statusCode = 400;

  return {
    statusCode,
    message: 'Validation Error',
    errorSources,
  };
};

export default handleValidationError;
```

- When user send wrong data in the params that time cast error occur. I will explain how to handle it. 

-Below i will explain the if else block for cast error in the globalErrorhandler.ts file and handleCastError.ts file

```javascript
// globalErrorhandler.ts

else if (err?.name === 'CastError') { // To find out cast error we are checking the name property of the err object and matching it with value of 'CastError'.
    const simplifiedError = handleCastError(err); // passing the err to the handleCastError function in the handleCastError.ts file
     statusCode = simplifiedError?.statusCode; // setting the status code from simplifiedError
    message = simplifiedError?.message;  // setting the message from simplifiedError
    errorSources = simplifiedError?.errorSources; // setting the error sources from simplifiedError
  }

// handleCastError.ts

import mongoose from 'mongoose';
import { TErrorSources, TGenericErrorResponse } from '../interface/error';

const handleCastError = (
  err: mongoose.Error.CastError,
): TGenericErrorResponse => { //Setting error and response type
  const errorSources: TErrorSources = [
    {
      path: err.path, // Getting the path from path property
      message: err.message, // Getting the message from message property
    },
  ];

// Following are same as zod validation / mongoose error
  const statusCode = 400;

  return {
    statusCode,
    message: 'Invalid ID',
    errorSources,
  };
};

export default handleCastError;
```
- Usually in order to prevent duplicate entry in the database we use pre hook in the model file. an example is as follows

```javascript
import httpStatus from 'http-status';
import { Schema, model } from 'mongoose';
import AppError from '../../errors/AppError';
import { TAcademicDepartment } from './academicDepartment.interface';

const academicDepartmentSchema = new Schema<TAcademicDepartment>(
  {
    name: {
      type: String,
      required: true,
      unique: true, // it creates an index in the database. So if it finds any value second time then it throw an error with 11000 code. 
    },
    academicFaculty: {
      type: Schema.Types.ObjectId,
      ref: 'AcademicFaculty',
    },
  },
  {
    timestamps: true,
  },
);

// Before saving the document into database following prehook check in the database whether any academic department exist in the database with the same name. If exist then using AppError it throw an error. If name does not exist it call next function to perform remaining task. 
academicDepartmentSchema.pre('save', async function (next) {
  const isDepartmentExist = await AcademicDepartment.findOne({
    name: this.name,
  });

  if (isDepartmentExist) {
    throw new AppError(
      httpStatus.NOT_FOUND,
      'This department is already exist!',
    );
  }

  next();
});

export const AcademicDepartment = model<TAcademicDepartment>(
  'AcademicDepartment',
  academicDepartmentSchema,
);
```

- So the 11000 error is thrown by unique property of the mongoose. We can use error handler to handle it. 

- -Below i will explain the if else block for cast error in the globalErrorhandler.ts file and handleDuplicateError.ts file 

```javascript
// globalErrorhandler.ts file

else if (err?.code === 11000) { // Checking whether the value of code property inside teh err object has 11000
    const simplifiedError = handleDuplicateError(err); // passing the err to the handleDuplicateError function in the handleDuplicateError.ts file
     statusCode = simplifiedError?.statusCode; // setting the status code from simplifiedError
    message = simplifiedError?.message;  // setting the message from simplifiedError
    errorSources = simplifiedError?.errorSources; // setting the error sources from simplifiedError
      }

// handleDuplicateError.ts file 

import { TErrorSources, TGenericErrorResponse } from '../interface/error';

const handleDuplicateError = (err: any): TGenericErrorResponse => {
  // Extract value within double quotes using regex
  const match = err.message.match(/"([^"]*)"/);

  // The extracted value will be in the first capturing group
  const extractedMessage = match && match[1];

  const errorSources: TErrorSources = [
    {
      path: '',
      message: `${extractedMessage} is already exists`,
    },
  ];

// Following are same as like earlier files.
  const statusCode = 400;

  return {
    statusCode,
    message: 'DUPLICATE ERROR',
    errorSources,
  };
};

export default handleDuplicateError;
```

- Adjusting AppError and throw new Error to keep consistancy

```javascript
else if (err instanceof AppError) {// Checking whether it is app error
    statusCode = err?.statusCode; // Getting statusCode from err and setting it in the statusCode
    message = err.message; // Getting message from err and setting it in the message
    errorSources = [// We are creating errorSource here
      {
        path: '', // As we cannot define the path of the error so we are keeping it as empty string.
        message: err?.message,
      },
    ];
  } else if (err instanceof Error) { // Checking whether it is error. 

  // There will be no status code in the err so the default will be used.
    message = err.message;// Getting message from err and setting it in the message
    errorSources = [// We are creating errorSource here
      {
        path: '',// As we cannot define the path of the error so we are keeping it as empty string.
        message: err?.message,
      },
    ];
  }
```

- Now we will see how to deal with UnhandledRejection and UncaughtException

- Lets see some theory. Node.js is a process. This process will sometime stop automatically and sometime we have to stop it using error handling. We cannot run server with error in it. If we use 1 inside the process.exit(1) then the process will stop. If we find any error in the synchronous code that means uncaughtException that time we will use process.exit(1) to stop the process. We will stop it immediately. In the asynchronous code as there are several process are running so close all the process first then close the server. This is called gracefully handle. 

We will handle it in the server.js file. The code will be as follows

```javascript
import { Server } from 'http';
import mongoose from 'mongoose';
import app from './app';
import config from './app/config';

let server: Server;  // this variable will holding a server and it is imported from the http.

async function main() {
  try {
    await mongoose.connect(config.database_url as string);

    server = app.listen(config.port, () => {
      console.log(`app is listening on port ${config.port}`);
    });  //here app.listen is a server so we assigned it to the server variable
  } catch (err) {
    console.log(err);
  }
}

main();

// Here in the process we are listening for the unhandledRejection 
process.on('unhandledRejection', () => {
  console.log(`😈 unahandledRejection is detected , shutting down ...`);
  if (server) { // First we will check that whether any application is running in hte server then we will stop the process and close the server
    server.close(() => {
      process.exit(1);
    });
  }
  process.exit(1); // if no process is running at the server we will stop the server immediately
});

// Here in the process we are listening for the uncaughtException 

process.on('uncaughtException', () => {
  console.log(`😈 uncaughtException is detected , shutting down ...`);
  process.exit(1); //If uncaughtException is found we will stop the process immediately. 
});
```