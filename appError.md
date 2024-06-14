## Creating a Custom Error Class in an express app

### Introduction

Error handling is a crucial aspect of developing robust applications. JavaScript provides a built-in "Error" class to throw exceptions, but it lacks the flexibility to include additional details like HTTP status codes. This blog post will guide you through creating a custom error class by extending the "Error" class, allowing you to provide more informative error responses.

- This is the tenth blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first nine blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide", "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function" and "Understanding Populating Referencing Fields in Mongoose". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-not-found-route-in-express-app-1d26

https://dev.to/md_enayeturrahman_2560e3/creating-a-custom-send-response-utility-function-in-express-2fg9

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-routes-in-an-express-app-a-step-by-step-guide-177j

https://dev.to/md_enayeturrahman_2560e3/simplifying-error-handling-in-express-controllers-introducing-catchasync-utility-function-2f3l

https://dev.to/md_enayeturrahman_2560e3/understanding-populating-referencing-fields-in-mongoose-jhg

### Basic Error Handling

In JavaScript, throwing an error is straightforward:

```javascript
throw new Error('This department already exists!');
```

This method only allows sending a simple error message. However, in many applications, especially web applications, it's essential to include more information, such as HTTP status codes, to make error handling more informative and user-friendly.

### Creating a Custom Error Class

To overcome the limitations of the basic "Error" class, we can create a custom error class that extends "Error". This custom class will allow us to add properties like statusCode to provide more context for the error.

Let's create a file named "AppError.ts" and define our custom error class:

```javascript
class AppError extends Error {
  public statusCode: number;

  constructor(statusCode: number, message: string, stack = '') {
    super(message);
    this.statusCode = statusCode;

    if (stack) {
      this.stack = stack;
    } else {
      Error.captureStackTrace(this, this.constructor);
    }
  }
}

export default AppError;
```

In this "AppError" class:

- We extend the built-in Error class.
- We add a statusCode property to hold the HTTP status code.
- We capture the stack trace, making debugging easier.

### Using the Custom Error Class

Now, let's use the AppError class in our academicDepartmentModel.ts file to throw more informative errors.

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
      unique: true,
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

academicDepartmentSchema.pre('save', async function (next) {
  const isDepartmentExist = await AcademicDepartment.findOne({
    name: this.name,
  });

  if (isDepartmentExist) {
    throw new AppError(
      httpStatus.NOT_FOUND,
      'This department already exists!',
    );
  }

  next();
});

academicDepartmentSchema.pre('findOneAndUpdate', async function (next) {
  const query = this.getQuery();
  const isDepartmentExist = await AcademicDepartment.findOne(query);

  if (!isDepartmentExist) {
    throw new AppError(
      httpStatus.NOT_FOUND,
      'This department does not exist!',
    );
  }

  next();
});

export const AcademicDepartment = model<TAcademicDepartment>(
  'AcademicDepartment',
  academicDepartmentSchema,
);
```

### Recap

To implement a custom error class in JavaScript, follow these steps:

- Create a Custom Error Class:
  - Extend the built-in Error class.
  - Add any additional properties needed (e.g., statusCode).

- Use the Custom Error Class:

  - Import and throw the custom error in your application to provide more informative error messages.

- Handle Errors Appropriately:

  - Catch these custom errors in your error-handling middleware or functions to send proper responses to the client.

### Conclusion

By extending the Error class, you can create more detailed and informative error messages, improving the overall robustness and user experience of your application.