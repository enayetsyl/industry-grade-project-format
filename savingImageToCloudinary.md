## How to Save Image with Multer

### Introduction 

- In any web app with user login functionality, there is usually a user image saving feature. The flow is as follows: We receive the user image from the frontend through the request body. We use Multer at the backend to save the file in a temporary folder on the server. From the folder, we save the image to Cloudinary or ImageBB, etc., and get a link. Then, we save the link into our database.

- Below, all the processes will be explained step by step.

### Install multer and cloudinary

- Run the following command in the server terminal:

```javascript
npm i multer cloudinary
```

### Getting necessary config data from the cloudinary

- Go to the cloudinary account. [Cloudinary](https://cloudinary.com)

- Sign up or log in. 

- On the left side, there are icons. If you hover over the second icon, it will display "Programmable Media." Click it. You will see a dashboard like the following image.
![Dashboard Image](https://res.cloudinary.com/deqyxkw0y/image/upload/v1719376995/programmagle-media_uykelr.jpg)

- Click the "Go to API Keys" button at the top right corner.

- Now click the "Generate New API Keys" button.

![Generate New API key button](https://res.cloudinary.com/deqyxkw0y/image/upload/v1719376994/secret_visible_o4bdbn.jpg)

- A new window will appear like the following image. Check your email. You will receive a code. Copy it, paste it here, and click the approve button. 

![Password Verification](https://res.cloudinary.com/deqyxkw0y/image/upload/v1719376994/password_verification_fgpwkc.jpg)

- Now you will see a new API key and secret added to the dashboard. At the top of the page, you will see "API Keys: Cloud name:..." copy it. In the lower middle of the image, you will see the API Key column under which you see the API key. Copy it. Click the eye button beside the API secret. A password verification window will appear. Go to your email; a new verification code will be sent there. Copy that, paste it here, and click the approve button.

![API Keys](https://res.cloudinary.com/deqyxkw0y/image/upload/v1719376994/api_key_yc9btu.jpg)

- Now you will see that the secret is visible as follows. Copy that. 

![Visible secret](https://res.cloudinary.com/deqyxkw0y/image/upload/v1719376994/secret_visible_o4bdbn.jpg)

- Now put the cloud name, api key and api secret in the .env file of your project. 

### utils file

- Below is the code of the utils file where the file storage using Multer, the file upload function to Cloudinary, and the file delete function from the folder are mentioned.

```javascript
import { UploadApiResponse, v2 as cloudinary } from 'cloudinary'; // Importing cloudinary for upload photo
import fs from 'fs'; // This will be required for file delete from temporary folder
import multer from 'multer'; // Multer import
import config from '../config'; // Importing config file so that the environmental variable can be used

// The config function set the cloud name, api key and secret from the config file. 

cloudinary.config({
  cloud_name: config.cloudinary_cloud_name,
  api_key: config.cloudinary_api_key,
  api_secret: config.cloudinary_api_secret,
});


// This is custom made function that is responsible for uploading image and after that deleting it.

export const sendImageToCloudinary = (
  imageName: string, // this is the name that will be used to save the image in the cloudinary
  path: string, // this is the path from where the file will be uploaded to the cloudinary.
): Promise<Record<string, unknown>> => { 
  return new Promise((resolve, reject) => { // Cloudinary does not provide async function so this custom make Promise is created to perform async task
    cloudinary.uploader.upload( // function provided by cloudinary to upload image
      path, // This is the path from where image will be uploaded. It is received through params. 
      { public_id: imageName.trim() }, // This is the image name that is received through params.
      function (error, result) {
        if (error) {
          reject(error); // If any error occur during upload that time the promise rejected. 
        }
        resolve(result as UploadApiResponse); // If upload is successful that time the result is returned. 
        // delete a file asynchronously
        fs.unlink(path, (err) => { // This function will delete the file from the temporary folder. It takes path as a params.
          if (err) {
            console.log(err);
          } else {
            console.log('File is deleted.');
          }
        });
      },
    );
  });
};

const storage = multer.diskStorage({ // This is the multer function to save the image received from the frontend to the temporary folder.
  destination: function (req, file, cb) { // this function is responsible for saving file into temporary folder. 
    cb(null, process.cwd() + '/uploads/');  // I named the temporary folder as uploads.
  },
  filename: function (req, file, cb) {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9); // This variable is holding the unique name of the file that will be stored in the uploads folder based on date adn some random number. 
    cb(null, file.fieldname + '-' + uniqueSuffix);
  },
});

export const upload = multer({ storage: storage });
```

### Routes File to Receive Data from the Frontend

- Below is the router.ts file that define the route and validate request.

```javascript
/* eslint-disable @typescript-eslint/no-explicit-any */
import express, { NextFunction, Request, Response } from 'express';  // Types imported from express
import validateRequest from '../../middlewares/validateRequest';  // Middleware function to validate data received from request
import { upload } from '../../utils/sendImageToCloudinary'; // Importing upload from utils that we created earlier.
import { createStudentValidationSchema } from '../Student/student.validation';  // Zod schema for validating data received from the frontend. 
import { UserControllers } from './user.controller'; // 

const router = express.Router();

router.post(
  '/create-student',  // route name
   upload.single('file'),  // Getting single file. you have to use the name 'file' at the frontend at the time of sending image. 
  (req: Request, res: Response, next: NextFunction) => { // Custom middleware. Usually from the frontend data passed to the backend in json format. But if you want to send image then you cannot send it json format. YOu have to use form-data, in the form data option you can send text as text format not json. This middleware is taking the text data from the req.body and converting it into json data and then assigning it again to the rew.body so that next middleware can use this json data to validate the data using zod. 
    req.body = JSON.parse(req.body.data);
    next();
  },
  validateRequest(createStudentValidationSchema), // Middleware to validate data using zod.
  UserControllers.createStudent, // Passing the data to the controller.
);
```

### Controller function 

- The controller function takes data from the route, passes it to the service function, and also receives data from the service function to send a response to the frontend. 

```javascript
import httpStatus from 'http-status'; // Packaged used for getting the status code
import catchAsync from '../../utils/catchAsync'; // Custom made catchAsync function to remove the use of try catch block 
import sendResponse from '../../utils/sendResponse'; // Custom made send response function. 
import { UserServices } from './user.service'; // Service function 

const createStudent = catchAsync(async (req, res) => {
  const { password, student: studentData } = req.body;  // Getting data from the req.body

  const result = await UserServices.createStudentIntoDB( // calling the service function and passing the file, password and data. 
    req.file,
    password,
    studentData,
  );

// Sending response to the frontend. 
  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Student is created succesfully',
    data: result,
  });
});
```

### Service function  function 

- This service function is responsible for saving user data into the database and calling the Cloudinary function. 

```javascript
import { sendImageToCloudinary } from '../../utils/sendImageToCloudinary';
import { TStudent } from '../Student/student.interface';
import { Student } from '../Student/student.model';
import { TUser } from './user.interface';
import { User } from './user.model';
import {  generateStudentId} from './user.utils';

const createStudentIntoDB = async ( // Getting param from the controller
  file: any,
  password: string,
  payload: TStudent,
) => {
  // create a user object
  const userData: Partial<TUser> = {}; // Creating a new object

   //set student role
  userData.role = 'student';  // Adding role to the userData object.
  // set student email
  userData.email = payload.email; // Adding email to the userData object.

  //set  generated id
  userData.id = await generateStudentId(admissionSemester); // Generating id and adding to the userData object.

  if (file) {
  const imageName = `${userData.id}${payload?.name?.firstName}`; // Creating file name from the user data.
  const path = file?.path; // Storing file path to path variable 

  //send image to cloudinary
  const { secure_url } = await sendImageToCloudinary(imageName, path);  // Passing image name and path to the cloudinary function and getting  secure url.

  userData.profileImg = secure_url as string; // Adding secure_url to the profile image

   }
 const result = await Student.create(userData)  // Creating new document in the Student collection.

 return result
};
```

### Conclusion

- One important point to mention you cannot use it in free vercel host. Because it does not allow to add temporary folder in the free instance. So you can check it in the localhost or using paid server. 

- Feel free to contact me at [linkedin](https://www.linkedin.com/in/md-enayetur-rahman)