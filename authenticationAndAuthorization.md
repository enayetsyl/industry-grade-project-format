## Authentication and authorization in an express app


### Definition

- Authentication answers the question: Who are you?
- Authorization answers the question: What are you allowed to do?


### Authentication

Authentication is the process of verifying the identity of a user or entity. It ensures that the person or system attempting to access a resource is who they claim to be. Authentication typically involves the following:

- User Credentials: Commonly, this involves a username and password. Other methods include biometric verification (fingerprint, retina scan), security tokens, smart cards, and multi-factor authentication (MFA) which combines two or more verification methods.

- Verification Process: The credentials provided by the user are checked against a database of authorized users. If the credentials match, the user is considered authenticated.

### Authorization
Authorization is the process of determining if an authenticated user has the right to access specific resources or perform certain actions. It deals with permissions and access controls.

- Access Control: Once a user is authenticated, the system checks what resources the user is allowed to access and what operations they are allowed to perform.

- Permission Levels: Different users or groups of users might have different levels of access. For example, an admin might have access to all parts of a system, while a regular user might only have access to their own data.

### Application of authentication

- Normally "auth" prefixed with an authentication route. e.g. /auth/login, /auth/signup 
- For authentication we need controller, interface, route, service, utils, validation etc files but not the  model file because we will work on user model for authentication. 

- We will begin our process with interface. An example of interface will be as follows

```javascript
export type TLoginUser = {
  id: string; 
  password: string;
};

// these two items we will receive from the frontend.
```
- We will also check the user interface file because it will be used later in the service file. Its content is as follows

```javascript
/* eslint-disable no-unused-vars */
import { Model } from 'mongoose';
import { USER_ROLE } from './user.constant';

// Change to interface from type because interface can be extended
export interface TUser {
  id: string;
  email: string;
  password: string;
  needsPasswordChange: boolean;
  passwordChangedAt?: Date;
  role: 'admin' | 'student' | 'faculty';
  status: 'in-progress' | 'blocked';
  isDeleted: boolean;
}

// TUser is extended to UserModel. The method defined the UserModel will be used at the time of creation of model and schema in the user model file.

export interface UserModel extends Model<TUser> {
  //instance methods for checking if the user exist
  isUserExistsByCustomId(id: string): Promise<TUser>;
  //instance methods for checking if passwords are matched
  isPasswordMatched(
    plainTextPassword: string,
    hashedPassword: string,
  ): Promise<boolean>;
  isJWTIssuedBeforePasswordChanged(
    passwordChangedTimestamp: Date,
    jwtIssuedTimestamp: number,
  ): boolean;
}

export type TUserRole = keyof typeof USER_ROLE;
```

### zod validation

- Then will come zod validation. Which is as follows:


```javascript
import { z } from 'zod';
// For login i am checking whether user is sending id and password
const loginValidationSchema = z.object({
  body: z.object({
    id: z.string({ required_error: 'Id is required.' }),
    password: z.string({ required_error: 'Password is required' }),
  }),
});

// For changing password i am checking whether user is sending old and new password. We put the requirement of giving old password. because if at the login state user left the pc unattended and someone else want to change the password that time he/she cannot do it because he/she cannot provide the old password. if we put the requirement that only new password should be given then any one can change password if find pc unattended.
const changePasswordValidationSchema = z.object({
  body: z.object({
    oldPassword: z.string({
      required_error: 'Old password is required',
    }),
    newPassword: z.string({ required_error: 'Password is required' }),
  }),
});

// For this is schema i am checking whether refreshToken is sent through cookies

const refreshTokenValidationSchema = z.object({
  cookies: z.object({
    refreshToken: z.string({
      required_error: 'Refresh token is required!',
    }),
  }),
});

// For forgot password i am checking whether user is sending id 
const forgetPasswordValidationSchema = z.object({
  body: z.object({
    id: z.string({
      required_error: 'User id is required!',
    }),
  }),
});

// For reset password i am checking whether user is sending id and new password
const resetPasswordValidationSchema = z.object({
  body: z.object({
    id: z.string({
      required_error: 'User id is required!',
    }),
    newPassword: z.string({
      required_error: 'User password is required!',
    }),
  }),
});

// exporting all the validation schema
export const AuthValidation = {
  loginValidationSchema,
  changePasswordValidationSchema,
  refreshTokenValidationSchema,
  forgetPasswordValidationSchema,
  resetPasswordValidationSchema,
};
```

### validate request middleware

- This is validateRequest.ts file. Is is a middleware that will be used to valued the req.body and req.cookies received from the frontend according to zod.

```javascript
import { NextFunction, Request, Response } from 'express';
import { AnyZodObject } from 'zod';
import catchAsync from '../utils/catchAsync';

const validateRequest = (schema: AnyZodObject) => {
  return catchAsync(async (req: Request, res: Response, next: NextFunction) => {
    await schema.parseAsync({
      body: req.body,
      cookies: req.cookies,
    });

    next();
  });
};

export default validateRequest;
```

### Auth.ts middleware 
- Now we will see the auth middleware file that will be used in the route file for authorization

```javascript
import { NextFunction, Request, Response } from 'express';
import httpStatus from 'http-status';
import jwt, { JwtPayload } from 'jsonwebtoken';
import config from '../config';
import AppError from '../errors/AppError';
import { TUserRole } from '../modules/user/user.interface';
import { User } from '../modules/user/user.model';
import catchAsync from '../utils/catchAsync';

const auth = (...requiredRoles: TUserRole[])  => { // a route can be accessed by user with different role. so in the route file when we will use auth guard we may provide multiple user role separated by comma. The ...requiredRoles (rest operator syntax) will combine them in an array.
  return catchAsync(async (req: Request, res: Response, next: NextFunction) => { // It is taking request, response and next function as a param. 
    const token = req.headers.authorization; // From request header we are taking the token

    // checking if the token is missing
    if (!token) {
      throw new AppError(httpStatus.UNAUTHORIZED, 'You are not authorized!');
    }

    // checking if the given token is valid
    const decoded = jwt.verify(
      token, // passing the token received from the frontend
      config.jwt_access_secret as string, // getting the secret from the index.ts file
    ) as JwtPayload;

    const { role, userId, iat } = decoded;  // destructuring the userId, role and iat from decoded token

    // checking if the user is exist
    const user = await User.isUserExistsByCustomId(userId);

    if (!user) {
      throw new AppError(httpStatus.NOT_FOUND, 'This user is not found !');
    }
    // checking if the user is already deleted

    const isDeleted = user?.isDeleted;

    if (isDeleted) {
      throw new AppError(httpStatus.FORBIDDEN, 'This user is deleted !');
    }

    // checking if the user is blocked
    const userStatus = user?.status;

    if (userStatus === 'blocked') {
      throw new AppError(httpStatus.FORBIDDEN, 'This user is blocked ! !');
    }

  // Checking whether the provided token is before password change. if it is before password change then we will not allow user to get data using it. 
    if (
      user.passwordChangedAt &&
      User.isJWTIssuedBeforePasswordChanged(
        user.passwordChangedAt,
        iat as number,
      )
    ) {
      throw new AppError(httpStatus.UNAUTHORIZED, 'You are not authorized !');
    }

  // Checking whether required role is present and if present then the role from decoded match with any of the required role. We created a new file called user.constant where the required role are kept and it will shown later. 
    if (requiredRoles && !requiredRoles.includes(role)) {
      throw new AppError(
        httpStatus.UNAUTHORIZED,
        'You are not authorized  hi!',
      );
    }

    // Adding user property in the request object and setting decoded as its value. in the index.d.ts file we modified the namespace of the Express to add it. 
    req.user = decoded as JwtPayload;
    next();
  });
};

export default auth;
```

### user.constant file

- Below is the content of user.constant file that was importer above auth.ts file

```javascript
export const USER_ROLE = {
  student: 'student',
  faculty: 'faculty',
  admin: 'admin',
} as const; // This object is set as const so it can not modify. Type for this object is set in the user.interface.ts file. for your easy understanding is is again given below

// export type TUserRole = keyof typeof USER_ROLE;

export const UserStatus = ['in-progress', 'blocked'];
```

### Extending the express type

- Below is the code of extending the request object of Express. Following code is written in the index.d.ts file. The purpose of it is to include the user property in the global namespace of Express Request type so that where ever the request type will be used it will have name property in it that has a type of JwtPayload.

```javascript
import { JwtPayload } from 'jsonwebtoken';

declare global {
  namespace Express {
    interface Request {
      user: JwtPayload;
    }
  }
}
```

### route file

- Now we will work on route file. Its content is as follows

```javascript
import express from 'express';
import auth from '../../middlewares/auth';
import validateRequest from '../../middlewares/validateRequest';
import { USER_ROLE } from './../user/user.constant';
import { AuthControllers } from './auth.controller';
import { AuthValidation } from './auth.validation';

const router = express.Router();

// This is login route, it validate request using zod then pass to the controller
router.post(
  '/login',
  validateRequest(AuthValidation.loginValidationSchema),
  AuthControllers.loginUser,
);

// This is change password route, it applies auth guard first (admin, faculty and student will have access to this route) then it validate request using zod then pass to the controller
router.post(
  '/change-password',
  auth(USER_ROLE.admin, USER_ROLE.faculty, USER_ROLE.student),
  validateRequest(AuthValidation.changePasswordValidationSchema),
  AuthControllers.changePassword,
);

// This is refresh token route, it does not applies auth guard but it validate request using zod then pass to the controller
router.post(
  '/refresh-token',
  validateRequest(AuthValidation.refreshTokenValidationSchema),
  AuthControllers.refreshToken,
);

// This is forget password route, it does not applies auth guard but it validate request using zod then pass to the controller. User will hit this route when they will forget password
router.post(
  '/forget-password',
  validateRequest(AuthValidation.forgetPasswordValidationSchema),
  AuthControllers.forgetPassword,
);

// This is reset-password route, it does not applies auth guard but it validate request using zod then pass to the controller
router.post(
  '/reset-password',
  validateRequest(AuthValidation.forgetPasswordValidationSchema),
  AuthControllers.resetPassword,
);

export const AuthRoutes = router;
```

### controller file

- New we will move to controller file. Its contents are as follows:

```javascript
import httpStatus from 'http-status';
import config from '../../config';
import catchAsync from '../../utils/catchAsync';
import sendResponse from '../../utils/sendResponse';
import { AuthServices } from './auth.service';

// This is the controller for login. 
const loginUser = catchAsync(async (req, res) => {
  const result = await AuthServices.loginUser(req.body); // It is passing user id and password to the service function
  const { refreshToken, accessToken, needsPasswordChange } = result; // Result received from service function are destructured here for further processing.

  res.cookie('refreshToken', refreshToken, {
    secure: config.NODE_ENV === 'production',
    httpOnly: true,
  }); //refresh token received from service is set to cookies

// accessToken and needsPasswordChange are sent to the frontend. you can set access token to the cookies also.
  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'User is logged in successfully!',
    data: {
      accessToken,
      needsPasswordChange,
    },
  });
});


const changePassword = catchAsync(async (req, res) => {
  const { ...passwordData } = req.body;  // Destructuring password data from req.body

  // I added user in the req object at auth component. Now i am passing the user to the service function so that it will understand which users password needs to change.
  const result = await AuthServices.changePassword(req.user, passwordData);

  // sending success message after changing the password.
  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Password is updated successfully!',
    data: result,
  });
});


const refreshToken = catchAsync(async (req, res) => {
  const { refreshToken } = req.cookies;  // destructuring refresh token from req.cookies
  const result = await AuthServices.refreshToken(refreshToken); // Passing refresh token to the service file. 

  // returning access token to the front end

  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Access token is retrieved successfully!',
    data: result, 
  });
});


const forgetPassword = catchAsync(async (req, res) => {
  const userId = req.body.id;  // extracting user id from req.body and passing it to the service function.
  const result = await AuthServices.forgetPassword(userId);

  // returning password reset link
  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Reset link is generated successfully!',
    data: result,
  });
});

const resetPassword = catchAsync(async (req, res) => {
  const token = req.headers.authorization;  // getting token from the header

  const result = await AuthServices.resetPassword(req.body, token); // passing token and req.body to the service function


  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Password reset successfully!',
    data: result,
  });
});

export const AuthControllers = {
  loginUser,
  changePassword,
  refreshToken,
  forgetPassword,
  resetPassword,
};
```

### user model file

- In the service file i will work on User model so lets look the user model file first before move on to service file.

```javascript
/* eslint-disable @typescript-eslint/no-this-alias */
import bcrypt from 'bcrypt';
import { Schema, model } from 'mongoose';
import config from '../../config';
import { UserStatus } from './user.constant';
import { TUser, UserModel } from './user.interface';

// Below is the schema for user. The UserModel is imported from the interface file that we created earlier. 
const userSchema = new Schema<TUser, UserModel>(
  {
    id: {
      type: String,
      required: true,
      unique: true,
    },
    email: {
      type: String,
      required: true,
      unique: true,
    },
    password: {
      type: String,
      required: true,
      select: 0,  //doing this will not send password to the frontend
    },
    needsPasswordChange: {
      type: Boolean,
      default: true,
    },
    passwordChangedAt: {
      type: Date,
    },
    role: {
      type: String,
      enum: ['student', 'faculty', 'admin'],
    },
    status: {
      type: String,
      enum: UserStatus,
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

// A pre hook is created to hash user's password before saving into database.
userSchema.pre('save', async function (next) {
  // eslint-disable-next-line @typescript-eslint/no-this-alias
  const user = this; // doc
  // hashing password and save into DB

  user.password = await bcrypt.hash(
    user.password,
    Number(config.bcrypt_salt_rounds),
  );

  next();
});

// A post hook is created so that after saving password when data sent to frontend the value of password field is set to empty string.
// set '' after saving password
userSchema.post('save', function (doc, next) {
  doc.password = '';
  next();
});

//A static is created so that it will take custom made id and find user based on it and include the password field of it.
userSchema.statics.isUserExistsByCustomId = async function (id: string) {
  return await User.findOne({ id }).select('+password');
};

// A static is created that will compare user given password with hashed password
userSchema.statics.isPasswordMatched = async function (
  plainTextPassword,
  hashedPassword,
) {
  return await bcrypt.compare(plainTextPassword, hashedPassword);
};

// A static is created that will compare the time to ensure that whether token is issued before password change so that token can be cancelled.
userSchema.statics.isJWTIssuedBeforePasswordChanged = function (
  passwordChangedTimestamp: Date,
  jwtIssuedTimestamp: number,
) {
  const passwordChangedTime =
    new Date(passwordChangedTimestamp).getTime() / 1000;
  return passwordChangedTime > jwtIssuedTimestamp;
};

// The UserModel is imported from the interface file that we created earlier. 
export const User = model<TUser, UserModel>('User', userSchema);
```

### utils file  
- Now we will understand the utils file where the function for token generation and verification created. They will be required in the service file.

```javascript
import jwt, { JwtPayload } from 'jsonwebtoken';

//This function is responsible for creating token. in the jwtPayload it receive userId and role, then secret and expire time also taken as a param. Using this it creates a token and return it.
export const createToken = (
  jwtPayload: { userId: string; role: string },
  secret: string,
  expiresIn: string,
) => {
  return jwt.sign(jwtPayload, secret, {
    expiresIn,
  });
};

// This is responsible for verifying token. It takes token and secret as param and check whether it is ok
export const verifyToken = (token: string, secret: string) => {
  return jwt.verify(token, secret) as JwtPayload;
};
```
 
 ### auth service file

- Now as we understand the user model and utils file contents, lets look at the service file. Below is the auth.service file content.

```javascript
import bcrypt from 'bcrypt';
import httpStatus from 'http-status';
import jwt, { JwtPayload } from 'jsonwebtoken';
import config from '../../config';
import AppError from '../../errors/AppError';
import { sendEmail } from '../../utils/sendEmail';
import { User } from '../user/user.model';
import { TLoginUser } from './auth.interface';
import { createToken, verifyToken } from './auth.utils';


const loginUser = async (payload: TLoginUser) => {
  // checking if the user is exist by passing user custom id to the static we created in the model file
  const user = await User.isUserExistsByCustomId(payload.id);

  // if no user is found using this custom id i am throwing error
  if (!user) {
    throw new AppError(httpStatus.NOT_FOUND, 'This user is not found !');
  }

  // checking if the user is already deleted

  const isDeleted = user?.isDeleted;

  if (isDeleted) {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is deleted !');
  }

  // checking if the user is blocked

  const userStatus = user?.status;

  if (userStatus === 'blocked') {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is blocked ! !');
  }

  //checking if the password is correct by calling the static method we created in the model file and passing the hashed password and user given password

  if (!(await User.isPasswordMatched(payload?.password, user?.password)))
    throw new AppError(httpStatus.FORBIDDEN, 'Password do not matched');

  //create token and sent to the  client

  const jwtPayload = {
    userId: user.id,
    role: user.role,
  };

  const accessToken = createToken(
    jwtPayload,
    config.jwt_access_secret as string,
    config.jwt_access_expires_in as string,
  );

  const refreshToken = createToken(
    jwtPayload,
    config.jwt_refresh_secret as string,
    config.jwt_refresh_expires_in as string,
  );

  return {
    accessToken,
    refreshToken,
    needsPasswordChange: user?.needsPasswordChange,
  };
};

// This function is responsible for password change
const changePassword = async (
  userData: JwtPayload,
  payload: { oldPassword: string; newPassword: string },
) => { // receiving user object, old and new password from controller
  // checking if the user is exist
  const user = await User.isUserExistsByCustomId(userData.userId);

  if (!user) {
    throw new AppError(httpStatus.NOT_FOUND, 'This user is not found !');
  }
  // checking if the user is already deleted

  const isDeleted = user?.isDeleted;

  if (isDeleted) {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is deleted !');
  }

  // checking if the user is blocked

  const userStatus = user?.status;

  if (userStatus === 'blocked') {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is blocked ! !');
  }

  //checking if the old password provided by the user is correct

  if (!(await User.isPasswordMatched(payload.oldPassword, user?.password)))
    throw new AppError(httpStatus.FORBIDDEN, 'Password do not matched');

  //hashing new password
  const newHashedPassword = await bcrypt.hash(
    payload.newPassword,
    Number(config.bcrypt_salt_rounds),
  );

  //Finding the document based on userId and role and passing new password, changing the needsPasswordChange field value to false as now password changed and adding password change time. It will be used cancel the  the access token issued before password change.
  await User.findOneAndUpdate(
    {
      id: userData.userId,
      role: userData.role,
    },
    {
      password: newHashedPassword,
      needsPasswordChange: false,
      passwordChangedAt: new Date(),
    },
  );

  return null;
};

const refreshToken = async (token: string) => {
  // checking if the given token is valid
  const decoded = verifyToken(token, config.jwt_refresh_secret as string);

  const { userId, iat } = decoded;  // destructuring user id and tat from the decoded

  // checking if the user is exist
  const user = await User.isUserExistsByCustomId(userId);

  if (!user) {
    throw new AppError(httpStatus.NOT_FOUND, 'This user is not found !');
  }
  // checking if the user is already deleted
  const isDeleted = user?.isDeleted;

  if (isDeleted) {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is deleted !');
  }

  // checking if the user is blocked
  const userStatus = user?.status;

  if (userStatus === 'blocked') {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is blocked ! !');
  }

// Checking whether token is issued before password change.
  if (
    user.passwordChangedAt &&
    User.isJWTIssuedBeforePasswordChanged(user.passwordChangedAt, iat as number)
  ) {
    throw new AppError(httpStatus.UNAUTHORIZED, 'You are not authorized !');
  }

  const jwtPayload = {
    userId: user.id,
    role: user.role,
  };

// Generating new access token
  const accessToken = createToken(
    jwtPayload,
    config.jwt_access_secret as string,
    config.jwt_access_expires_in as string,
  );

  return {
    accessToken,
  };
};


const forgetPassword = async (userId: string) => {
  // checking if the user is exist
  const user = await User.isUserExistsByCustomId(userId);

  if (!user) {
    throw new AppError(httpStatus.NOT_FOUND, 'This user is not found !');
  }
  // checking if the user is already deleted
  const isDeleted = user?.isDeleted;

  if (isDeleted) {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is deleted !');
  }

  // checking if the user is blocked
  const userStatus = user?.status;

  if (userStatus === 'blocked') {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is blocked ! !');
  }

  const jwtPayload = {
    userId: user.id,
    role: user.role,
  };

  // Creating a new token
  const resetToken = createToken(
    jwtPayload,
    config.jwt_access_secret as string,
    '10m',
  );

  // Creating a new link. It has three part. First part is front end base address. second part is user id and third part is newly generated token. 
  const resetUILink = `${config.reset_pass_ui_link}?id=${user.id}&token=${resetToken} `;

  // calling the email function to send the link to the user email. The sendEmail utility function will be shown later.
  sendEmail(user.email, resetUILink);

  console.log(resetUILink);
};




const resetPassword = async (
  payload: { id: string; newPassword: string },
  token: string,
) => { // getting user id, new password and token from controller
  // checking if the user is exist
  const user = await User.isUserExistsByCustomId(payload?.id);

  if (!user) {
    throw new AppError(httpStatus.NOT_FOUND, 'This user is not found !');
  }
  // checking if the user is already deleted
  const isDeleted = user?.isDeleted;

  if (isDeleted) {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is deleted !');
  }

  // checking if the user is blocked
  const userStatus = user?.status;

  if (userStatus === 'blocked') {
    throw new AppError(httpStatus.FORBIDDEN, 'This user is blocked ! !');
  }

  // verifying the token

  const decoded = jwt.verify(
    token,
    config.jwt_access_secret as string,
  ) as JwtPayload;

  //localhost:3000?id=A-0001&token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJBLTAwMDEiLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MDI4NTA2MTcsImV4cCI6MTcwMjg1MTIxN30.-T90nRaz8-KouKki1DkCSMAbsHyb9yDi0djZU3D6QO4

  // checking the id given by the user in the req.body and id inside the token does match .

  if (payload.id !== decoded.userId) {
    console.log(payload.id, decoded.userId);
    throw new AppError(httpStatus.FORBIDDEN, 'You are forbidden!');
  }

  //hash new password
  const newHashedPassword = await bcrypt.hash(
    payload.newPassword,
    Number(config.bcrypt_salt_rounds),
  );

  // updating new password to the db
  await User.findOneAndUpdate(
    {
      id: decoded.userId,
      role: decoded.role,
    },
    {
      password: newHashedPassword,
      needsPasswordChange: false,
      passwordChangedAt: new Date(),
    },
  );
};

export const AuthServices = {
  loginUser,
  changePassword,
  refreshToken,
  forgetPassword,
  resetPassword,
};
```

### nodemailer utility function

- For sending email to the user nodemailer package is used. Details config of the nodemailer will be discussed in a separate blog. Below just sendEmail function is shown

```javascript
import nodemailer from 'nodemailer';
import config from '../config';

export const sendEmail = async (to: string, html: string) => {
  const transporter = nodemailer.createTransport({
    host: 'smtp.gmail.com.',  // if you use gmail
    port: 587, // if you use gmail
    secure: config.NODE_ENV === 'production', // for only production 
    auth: {
      // pass should be collect from the gmail account through config
      user: 'mezbaul@programming-hero.com',
      pass: 'xfqj dshz wdui ymtb',
    },
  });

  await transporter.sendMail({
    from: 'mezbaul@programming-hero.com', // sender address
    to, // list of receivers, will come from param
    subject: 'Reset your password within ten mins!', // Subject line
    text: '', // plain text body
    html, // html body, will come from param
  });
};

```
