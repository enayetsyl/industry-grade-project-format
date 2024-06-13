## How to create API in an industry standard app

### Introduction 

- This is the third blog of my series where i am writing how to write code for industry grade project so that you can manage and scale the project. In this blog we will learn how to create api endpoint. We will see how to create interface, mongoose model, route, controller, service file and validate with zod. 

- The first two blogs of the series was about "How to set up eslint and prettier in a express and typescript project" and "Folder structure in an industry standard project". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

- Today's code will be written on top of them.

- Let's understand the main files and what are we going to do. The routes we will create will be for user. We will have an user.interface.ts file that will hold code for interface. Then we will have user.model.ts file that will contain the mongoose schema and model for the user. Then we will have user.validation.ts file. Here we will validate the data received from frontend using zod. After that we will have user.route.ts file that will contain code related to route. Then comes user.controller.ts file that will contains function to handle route logic and at last user.service.ts file that will contain the business logic of the controller code. 

- In order to get benefitted from the blog you have to go through all the code. I will write explanation of the code in the comment beside the code. 

### Folder structure

- At the beginning lets see the folder structure related to user Route.

```javascript
my-express-app/
│
├── .env
├── .eslintignore
├── .eslintrc.json
├── .gitignore
├── .prettierrc.json
├── package.json
├── tsconfig.json
├── node_modules/
│
├── src/
│   ├── app/
│   │   ├── middleware/
│   │   │   ├── auth.ts
│   │   │   ├── globalErrorhandler.ts
│   │   │   ├── notFound.ts
│   │   │   └── validateRequest.ts
│   │   ├── modules/
│   │   │   ├── Student/
│   │   │   │   ├── StudentConstant.ts
│   │   │   │   ├── StudentController.ts
│   │   │   │   ├── StudentInterface.ts
│   │   │   │   ├── StudentModel.ts
│   │   │   │   ├── StudentRoute.ts
│   │   │   │   └── StudentValidation.ts
│   │   │   ├── User/
│   │   │   │   ├── UserConstant.ts
│   │   │   │   ├── UserController.ts
│   │   │   │   ├── UserInterface.ts
│   │   │   │   ├── UserModel.ts
│   │   │   │   ├── UserRoute.ts
│   │   │   │   └── UserValidation.ts
│   │   ├── routes/
│   │   │   └── index.ts
│   │   ├── utils/
│   │   │   ├── catchAsync.ts
│   │   │   └── sendResponse.ts
│   ├── app.ts
│   └── server.js
```

- Above is the file and folders necessary for the creation of user route. For full file and folder structure please refer to the second blog of this series.

### User interface

- In our project, we have defined a user type in TypeScript named TUser. Although the file is named "interface," we are using a type declaration instead of an interface. Here's the definition:

```javascript
export type TUser = {
  id: string;
  password: string;
  needsPasswordChange: boolean;
  role: 'admin' | 'student' | 'faculty';
  status: 'in-progress' | 'blocked';
  isDeleted: boolean;
};
```
- Naming Convention: The type is named TUser, with the "T" prefix indicating it is a type. This is a convention to help differentiate types from other constructs in the code.

- **Properties:**

  - **id:** A string that uniquely identifies the user.

  - **password:** The user's password, stored as a string.

  - **needsPasswordChange:** A boolean indicating whether the user is required to change their password.

  - **role:** An enum-like property that specifies the user's role. In our app, there are three types of users: 'admin', 'student', and 'faculty'.

  - **status:** An enum-like property representing the user's current status, with possible values: 'in-progress' or 'blocked'. If a user's status is 'blocked', they cannot log in regardless of their role (admin, student, faculty). Authentication checks are performed on the user collection, so changing a user's status to 'blocked' here will prevent them from logging in, simplifying user management and maintenance.

  - **isDeleted:** A boolean that indicates whether the user has been deleted. This field is stored in the real database. No document is ever truly deleted from the database; instead, its "**isDeleted**" status is changed to "**true**". If "**isDeleted**" is "**false**", the user object will be sent to the frontend during a get request. If "**isDeleted**" is "**true**", the user object, although present in the database, will not be sent to the frontend during a get request.

### User Model

- The "user.model.ts" file defines the Mongoose schema for the user, including two hooks: pre-save and post-save. 

```javascript
import bcrypt from 'bcrypt';
import { Schema, model } from 'mongoose';
import config from '../../config';
import { TUser } from './user.interface';

const userSchema = new Schema<TUser>(
  {
    id: {
      type: String,
      required: true,
    },
    password: {
      type: String,
      required: true,
    },
    needsPasswordChange: {
      type: Boolean,
      default: true,
    },
    role: {
      type: String,
      enum: ['student', 'faculty', 'admin'],
    },
    status: {
      type: String,
      enum: ['in-progress', 'blocked'],
      default: 'in-progress',
    },
    isDeleted: {
      type: Boolean,
      default: false,
    },
  },
  {
    timestamps: true,
  },
);

userSchema.pre('save', async function (next) {
  
  const user = this; 
  // hashing password and save into DB
  user.password = await bcrypt.hash(
    user.password,
    Number(config.bcrypt_salt_rounds),
  );
  next();
});

// set '' after saving password
userSchema.post('save', function (doc, next) {
  doc.password = '';
  next();
});

export const User = model<TUser>('User', userSchema);
```

- **Imports:** I imported "bcrypt" to hash the password before saving it to the database. The "Schema" and "model" are imported from Mongoose. The "config" is imported from the index file inside the config folder, which holds the .env file variables (for details, see my first blog). The "TUser" type is imported from the interface file. It is passed to the schema to ensure that any deviations from the defined type during schema creation will trigger a warning.

- **Schema Definition:** The schema is defined using the **"Schema"** constructor from Mongoose, with **"TUser"** passed as a generic type to ensure type safety.

- **Fields:**

  -**id:** A string that uniquely identifies the user. This field is required.

  - **password:** The user's password, stored as a string. This field is required.

  - **needsPasswordChange:** A boolean indicating whether the user needs to change their password. It defaults to true.

  - **role:** A string that specifies the user's role. It can be 'student', 'faculty', or 'admin'.

  - **status:** A string representing the user's current status. It can be 'in-progress' or 'blocked', with a default value of 'in-progress'.

  - **isDeleted:** A boolean indicating whether the user has been deleted. It defaults to false.

- **Options:**

  - **timestamps:** When set to true, Mongoose will automatically add "**createdAt**" and "**updatedAt**" fields to the schema.

- **Explanation of the Pre-Save Hook:** The** pre('save')** hook in Mongoose is a middleware function that runs before a document is saved to the database. This pre-save hook ensures that the user's password is always hashed before being stored in the database, enhancing security by never storing plain-text passwords. Here's a breakdown of how it works in the **userSchema:**

  - **Pre-Save Hook:** The pre('save') function is a middleware that is executed before the save operation.

  - **Context Binding:** const user = this;: The this keyword refers to the document being saved. This line assigns this to user for clarity and to avoid ESLint warnings.

  - **Password Hashing:** user.password = await bcrypt.hash(user.password, Number(config.bcrypt_salt_rounds));: This line hashes the user's password using bcrypt before saving it to the database. The config.bcrypt_salt_rounds specifies the number of salt rounds used by bcrypt to generate the hash, enhancing password security.

  - **Calling next():** The next function is called to proceed with the save operation. Without calling next(), the save operation would be halted.

- **Explanation of the Post-Save Hook:** The post('save') hook in Mongoose is a middleware function that runs after a document has been saved to the database. Here's a breakdown of how it works in the userSchema:

  - **Post-Save Hook:** The **post('save')** function is a middleware that is executed after the save operation.

  - ** Setting Password to Empty String:** After saving a user document to the database, it's common practice to send the user data to the frontend as a response. However, for security reasons, we should avoid transmitting hashed passwords to the frontend. Despite being securely stored in the database, hashed passwords should remain confidential. Therefore, this hook ensures that the password field is set to an empty string before sending the user document to the frontend. By doing so, we prevent the transmission of sensitive information and uphold the security of our application.

  - **Calling next():** The next function is called to proceed after executing the hook. Without calling next(), the middleware chain would not continue.

- **Exporting user model:**  This line exports the Mongoose model named User, which is created using the model function provided by Mongoose. The model is defined based on the TUser type and the userSchema schema. This allows us to interact with the User collection in the database using methods provided by Mongoose, such as find, findOne, create, update, and delete.

### Validation using zod

- The user.validation.ts is dedicated to validating the password field only, employing a simple validation approach. This is a simple validation. I will write another blog that will detail various type of zod validation. 

```javascript
import { z } from 'zod';

const userValidationSchema = z.object({
  pasword: z
    .string({
      invalid_type_error: 'Password must be string',
    })
    .max(20, { message: 'Password can not be more than 20 characters' })
    .optional(),
});

export const UserValidation = {
  userValidationSchema,
};
```

- We import the z object from the Zod library.

- We define a validation schema for user data using Zod's object method.

- Within the schema, we define validation rules for the password field:

  - We specify that the password must be a string and provide a custom error message if the value is not a string.

  - We set a maximum length of 20 characters for the password and provide a custom error message if the length exceeds this limit.

  - The user will be created by the admin. At the time of user creation admin can send password. If admin does not send password then default password will be applied at teh backend. That is why We mark the password field as optional.

- Finally, we export the user validation schema as UserValidation.

- Now question comes that in the User model we can see there are several properties of the user (id, password, needsPasswordChange, role, status and isDeleted) but why we are validating the password field only.

- The id property will be unique and generated at the backend using auto increment method. So it need not come from frontend. So doesn't require validation. 

- The default value for needsPasswordChange, status and isDeleted fields are set in the type within the user.interface.ts file. So it need not come from frontend. So doesn't require validation. 

- The role will be set from the end point. So it also does not need to come from frontend. So doesn't require validation. 

- So the only field that may need to come from frontend is password. That is why we are only validating it even though the user object has other fields. 


### Constant file

- In our application, users can have one of three roles: student, faculty, or admin. To maintain a cleaner codebase and ensure consistency, we have created a separate file named "user.constant.ts" to hold these user roles as constants. Here's how it looks:

```javascript
export const USER_ROLE = {
  student: 'student',
  faculty: 'faculty',
  admin: 'admin',
} as const;
```
- These constants can then be imported and used in other files, such as "user.route.ts", making our code more organized and easier to maintain.

### Route file

- Our route file holds routes, connection with controller and application of middleware for verifying the admin privileges.

```javascript
import express from 'express';
import auth from '../../middlewares/auth';
import validateRequest from '../../middlewares/validateRequest';
import { createAdminValidationSchema } from '../Admin/admin.validation';
import { createFacultyValidationSchema } from '../Faculty/faculty.validation';
import { createStudentValidationSchema } from './../student/student.validation';
import { USER_ROLE } from './user.constant';
import { UserControllers } from './user.controller';

const router = express.Router();

// Route for creating a student
router.post(
  '/create-student',
  auth(USER_ROLE.admin), // Middleware to verify admin privileges
  validateRequest(createStudentValidationSchema), // Middleware for validating request
  UserControllers.createStudent, // Controller function for handling the request
);

// Route for creating a faculty member
router.post(
  '/create-faculty',
  auth(USER_ROLE.admin), // Middleware to verify admin privileges
  validateRequest(createFacultyValidationSchema), // Middleware for validating request
  UserControllers.createFaculty, // Controller function for handling the request
);

// Route for creating an admin user
router.post(
  '/create-admin',
  validateRequest(createAdminValidationSchema), // Middleware for validating request
  UserControllers.createAdmin, // Controller function for handling the request
);

export const UserRoutes = router;
```

- We define routes for creating students, faculty members, and admin users.

- Middleware functions are applied to ensure that only admin users can access the routes for creating students and faculty members.

- Request validation middleware is applied to validate the request body before passing it to the controller functions.

- Finally, the respective controller functions are invoked to handle the requests and perform the necessary actions.

### Controller file

- Below, I'll demonstrate two controller files. The first one utilizes a try-catch block for error handling, while the second one employs reusable code for error handling using a custom catchAsync function. I will write a separate blog for this reuseable code for try catch later. In this blog let's focus on the logic other than the try-catch blocks: 

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
- **httpStatus:** This package helps send status codes with responses by typing the response type.

- **createStudent Function:**

  - Takes three parameters: req, res, and next.

  - Destructures password and studentData from req.body.

  - Calls createStudentIntoDB with the password and student data.

  - Uses sendResponse to send the response to the frontend.

  - In the catch block, it calls next with the error.

```javascript
import httpStatus from 'http-status';
import catchAsync from '../../utils/catchAsync';
import sendResponse from '../../utils/sendResponse';
import { UserServices } from './user.service';

const createStudent = catchAsync(async (req, res) => {
  const { password, student: studentData } = req.body;

  const result = await UserServices.createStudentIntoDB(password, studentData);

  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Student is created succesfully',
    data: result,
  });
});

const createFaculty = catchAsync(async (req, res) => {
  const { password, faculty: facultyData } = req.body;

  const result = await UserServices.createFacultyIntoDB(password, facultyData);

  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Faculty is created succesfully',
    data: result,
  });
});

const createAdmin = catchAsync(async (req, res) => {
  const { password, admin: adminData } = req.body;

  const result = await UserServices.createAdminIntoDB(password, adminData);

  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Admin is created succesfully',
    data: result,
  });
});

export const UserControllers = {
  createStudent,
  createFaculty,
  createAdmin,
};
```

- The purpose of this code is similar to the previous one but with a different approach.

- **catchAsync:** A custom function to handle errors, eliminating the need for repetitive try-catch blocks.

- createStudent, createFaculty, createAdmin Functions:

  - Destructure data from req.body.
  - Call the respective service functions to create users.
  - Use sendResponse to send the response to the frontend.

- This approach removes the need for try-catch blocks and the next function for error handling, making the code cleaner and more reusable.

- These examples demonstrate how to structure controller functions for user creation while maintaining clean and manageable error handling. In a later blog, we will delve into creating a custom sendResponse function and managing errors using the next function


### Service file

- Below is the explanation of the user.service.ts file, which contains functions to create different types of users (students, faculty, and admins) in the database. Each function uses MongoDB transactions to ensure data consistency.

```javascript
import httpStatus from 'http-status';
import mongoose from 'mongoose';
import config from '../../config';
import AppError from '../../errors/AppError';
import { TAdmin } from '../Admin/admin.interface';
import { Admin } from '../Admin/admin.model';
import { TFaculty } from '../Faculty/faculty.interface';
import { Faculty } from '../Faculty/faculty.model';
import { AcademicDepartment } from '../academicDepartment/academicDepartment.model';
import { TStudent } from '../student/student.interface';
import { Student } from '../student/student.model';
import { AcademicSemester } from './../academicSemester/academicSemester.model';
import { TUser } from './user.interface';
import { User } from './user.model';
import {
  generateAdminId,
  generateFacultyId,
  generateStudentId,
} from './user.utils';

const createStudentIntoDB = async (password: string, payload: TStudent) => {
  // create a user object
  const userData: Partial<TUser> = {};

  //if password is not given , use deafult password
  userData.password = password || (config.default_password as string);

  //set student role
  userData.role = 'student';

  // find academic semester info
  const admissionSemester = await AcademicSemester.findById(
    payload.admissionSemester,
  );

  if (!admissionSemester) {
    throw new AppError(400, 'Admission semester not found');
  }

  const session = await mongoose.startSession();

  try {
    session.startTransaction();
    //set  generated id
    userData.id = await generateStudentId(admissionSemester);

    // create a user (transaction-1)
    const newUser = await User.create([userData], { session }); // array

    //create a student
    if (!newUser.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create user');
    }
    // set id , _id as user
    payload.id = newUser[0].id;
    payload.user = newUser[0]._id; //reference _id

    // create a student (transaction-2)

    const newStudent = await Student.create([payload], { session });

    if (!newStudent.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create student');
    }

    await session.commitTransaction();
    await session.endSession();

    return newStudent;
  } catch (err: any) {
    await session.abortTransaction();
    await session.endSession();
    throw new Error(err);
  }
};

const createFacultyIntoDB = async (password: string, payload: TFaculty) => {
  // create a user object
  const userData: Partial<TUser> = {};

  //if password is not given , use deafult password
  userData.password = password || (config.default_password as string);

  //set student role
  userData.role = 'faculty';

  // find academic department info
  const academicDepartment = await AcademicDepartment.findById(
    payload.academicDepartment,
  );

  if (!academicDepartment) {
    throw new AppError(400, 'Academic department not found');
  }

  const session = await mongoose.startSession();

  try {
    session.startTransaction();
    //set  generated id
    userData.id = await generateFacultyId();

    // create a user (transaction-1)
    const newUser = await User.create([userData], { session }); // array

    //create a faculty
    if (!newUser.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create user');
    }
    // set id , _id as user
    payload.id = newUser[0].id;
    payload.user = newUser[0]._id; //reference _id

    // create a faculty (transaction-2)

    const newFaculty = await Faculty.create([payload], { session });

    if (!newFaculty.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create faculty');
    }

    await session.commitTransaction();
    await session.endSession();

    return newFaculty;
  } catch (err: any) {
    await session.abortTransaction();
    await session.endSession();
    throw new Error(err);
  }
};

const createAdminIntoDB = async (password: string, payload: TAdmin) => {
  // create a user object
  const userData: Partial<TUser> = {};

  //if password is not given , use deafult password
  userData.password = password || (config.default_password as string);

  //set student role
  userData.role = 'admin';

  const session = await mongoose.startSession();

  try {
    session.startTransaction();
    //set  generated id
    userData.id = await generateAdminId();

    // create a user (transaction-1)
    const newUser = await User.create([userData], { session });

    //create a admin
    if (!newUser.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create admin');
    }
    // set id , _id as user
    payload.id = newUser[0].id;
    payload.user = newUser[0]._id; //reference _id

    // create a admin (transaction-2)
    const newAdmin = await Admin.create([payload], { session });

    if (!newAdmin.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create admin');
    }

    await session.commitTransaction();
    await session.endSession();

    return newAdmin;
  } catch (err: any) {
    await session.abortTransaction();
    await session.endSession();
    throw new Error(err);
  }
};

export const UserServices = {
  createStudentIntoDB,
  createFacultyIntoDB,
  createAdminIntoDB,
};
```

- **Create Student into DB:**  

  - **userData:** An object to hold user data.

  - **password:** Uses the provided password from the frontend or if no password is provided from the frontend a default one is used.

  - **role:** Sets the role to 'student' elevating the to send it from frontend.

  - **admissionSemester:** Fetches the admission semester from the database.

  - **session:** Starts a MongoDB session for transactions.

  - **Transaction:** Generates a student ID using generateStudentId function that will create automatically a unique id for new student.
    - Creates a user.
    - Sets the student ID and references the user ID.
    - Creates a student.
  - **Error Handling:** Uses try-catch for transaction management.

- **Create Faculty into DB:**
  
  - Similar to createStudentIntoDB, but for faculty members.

  - **role:** Sets the role to 'faculty'.
  - **academicDepartment:** Fetches the academic department from the database.

  - **Transaction:**
    - Generates a faculty ID.
    - Creates a user.
    - Sets the faculty ID and references the user ID.
    - Creates a faculty.

- **Create Admin into DB**

  - Similar to createStudentIntoDB and createFacultyIntoDB, but for admin users.
  - **role:** Sets the role to 'admin'.
  - **Transaction:**
    - Generates an admin ID.
    - Creates a user.
    - Sets the admin ID and references the user ID.
    - Creates an admin.

- Exports the user services for use in other parts of the application.

### Summary

- This blog demonstrate how to organize code for a specific route/collection that can be manageable and scalable. 