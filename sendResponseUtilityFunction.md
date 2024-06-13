## Creating a Custom Send Response Utility Function in Express

### Introduction

- When building an Express application, it is essential to have consistent and standardized responses to client requests. This not only enhances the user experience but also simplifies debugging and maintenance. In this blog, we'll walk through creating a custom send response utility function and demonstrate how to use it in your Express controllers.

- This is the sixth  blog of my series where i am writing how to write code for industry grade project so that you can manage and scale the project.  

- The first five blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express" and "How to handle not found route in express app". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-not-found-route-in-express-app-1d26

### The Send Response Utility Function

The send response utility function is designed to streamline the process of sending consistent JSON responses from your Express controllers. Here's the implementation:

```javascript
import { Response } from 'express';

type TResponse<T> = {
  statusCode: number;
  success: boolean;
  message?: string;
  data: T;
};

const sendResponse = <T>(res: Response, data: TResponse<T>) => {
  res.status(data?.statusCode).json({
    success: data.success,
    message: data.message,
    data: data.data,
  });
};

export default sendResponse;

```

**Explanation**
**Type Definition:** We define a generic type TResponse<T> to describe the shape of the response object. This includes the status code, success flag, an optional message, and the data to be sent.

**Function Definition:** The sendResponse function takes two parameters:
  - res: The Express response object.
  - data: An object conforming to the TResponse type.

**Response Handling:** Inside the function, we use the status method on the response object to set the HTTP status code. Then, we use the json method to send a JSON response containing the success flag, message, and data.

**Using the Send Response Utility Function in Controllers**

Now, let's see how to use this utility function in a typical Express controller. Here is an example with a createStudent function:

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

    const result = await UserServices.createStudentIntoDB(
      password,
      studentData,
    );

    sendResponse(res, {
      statusCode: httpStatus.OK,
      success: true,
      message: 'Student is created successfully',
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

**Explanation**

  - **Imports:** We import necessary modules and utilities, including httpStatus for HTTP status codes, Express types, and our sendResponse utility.

  - **Controller Function:** The createStudent function is an asynchronous function that handles creating a new student. It takes three parameters:
    - **req:** The Express request object.
    - **res:** The Express response object.
    - **next:** The next middleware function.

  - **Destructuring Request Body:** We destructure the password and student data from the request body.

  - **Service Call:** We call the createStudentIntoDB method from UserServices to handle the database logic. This function returns the created student data.

  - **Send Response:** We use the sendResponse utility function to send a JSON response. We pass the response object and an object containing the status code, success flag, message, and data.

- **Error Handling:** If an error occurs, we pass it to the next middleware function using next(err). This will typically be caught by a global error handler in the application.

### Conclusion

By creating and using a send response utility function, you can standardize your API responses, making your Express application more robust and easier to maintain. This approach ensures that all responses follow a consistent format, improving both development efficiency and user experience.

Feel free to integrate this pattern into your projects and adapt it to fit your specific needs. Stay tuned for more tips and best practices for building scalable and maintainable web applications!






