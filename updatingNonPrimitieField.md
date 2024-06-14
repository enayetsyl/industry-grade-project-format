## Updating Non-Primitive Data Dynamically in Mongoose

### Introduction 

When working with MongoDB and Mongoose, updating documents is straightforward for primitive fields. However, handling nested or non-primitive fields requires a more nuanced approach to ensure that the data is updated correctly without overwriting existing fields. In this blog post, we'll explore how to dynamically update non-primitive fields using a comprehensive example.

- This is the twelfth blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first eleven blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide", "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function", "Understanding Populating Referencing Fields in Mongoose", "Creating a Custom Error Class in an express app" and "Understanding Transactions and Rollbacks in MongoDB". You can check them in the following link.

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

### Understanding Primitive Field Updates

Let's start with a simple example of updating primitive fields. Suppose we have a user document:

```javascript
{
  name: 'Enayet',
  age: 36,
  male: true,
}
```
Updating a primitive field, such as age, is straightforward. If we send the new data:
```javascript
{
  age: 37
}
```
The document will be updated as follows:
```javascript
{
  name: 'Enayet',
  age: 37,
  male: true,
}
```
Similarly, updating multiple primitive fields, like name and male, is equally simple:
```javascript
{
  name: 'Mariyam',
  male: false,
}
```
The updated document will be:
```javascript
{
  name: 'Mariyam',
  age: 36,
  male: false,
}
```

### Challenges with Non-Primitive Fields

Updating non-primitive fields, such as nested objects, requires careful handling to avoid overwriting existing data. Consider the following document with a nested name object:

```javascript
{
  name: {
    firstName: 'Enayet',
    lastName: 'Rahman'
  },
  age: 36,
  male: true,
}
```
If we want to add a middleName property and send the data as follows:

```javascript
{
  name: {
    middleName: 'Nai'
  }
}
```
The updated document will be:
```javascript
{
  name: {
    middleName: 'Nai'
  },
  age: 36,
  male: true,
}
```
This approach overwrites the entire name object, removing firstName and lastName. To add middleName without losing other fields, we need to structure the data differently:

```javascript
name.middleName: 'Nai'
```

The resulting document will be:

```javascript
{
  name: {
    firstName: 'Enayet',
    middleName: 'Nai',
    lastName: 'Rahman'
  },
  age: 36,
  male: true,
}
```
Instead of sending data in this format from the frontend, we can send it as an object and handle the update logic on the backend.

### Comprehensive Example: Student Data

Let's use a comprehensive example to illustrate the solution. We have a Student type with various nested types:

- TypeScript Types

```javascript
import { Model, Types } from 'mongoose';

export type TUserName = {
  firstName: string;
  middleName: string;
  lastName: string;
};

export type TGuardian = {
  fatherName: string;
  fatherOccupation: string;
  fatherContactNo: string;
  motherName: string;
  motherOccupation: string;
  motherContactNo: string;
};

export type TLocalGuardian = {
  name: string;
  occupation: string;
  contactNo: string;
  address: string;
};

export type TStudent = {
  id: string;
  user: Types.ObjectId;
  password: string;
  name: TUserName;
  gender: 'male' | 'female' | 'other';
  dateOfBirth?: Date;
  email: string;
  contactNo: string;
  emergencyContactNo: string;
  bloodGroup?: 'A+' | 'A-' | 'B+' | 'B-' | 'AB+' | 'AB-' | 'O+' | 'O-';
  presentAddress: string;
  permanentAddress: string;
  guardian: TGuardian;
  localGuardian: TLocalGuardian;
  profileImg?: string;
  admissionSemester: Types.ObjectId;
  isDeleted: boolean;
  academicDepartment: Types.ObjectId;
};
```
- Mongoose Schema and Model

```javascript
import { Schema, model } from 'mongoose';
import { TStudent, TUserName, TGuardian, TLocalGuardian } from './student.interface';

const userNameSchema = new Schema<TUserName>({
  firstName: {
    type: String,
    required: [true, 'First Name is required'],
    trim: true,
    maxlength: [20, 'Name cannot be more than 20 characters'],
  },
  middleName: {
    type: String,
    trim: true,
  },
  lastName: {
    type: String,
    trim: true,
    required: [true, 'Last Name is required'],
    maxlength: [20, 'Name cannot be more than 20 characters'],
  },
});

const guardianSchema = new Schema<TGuardian>({
  fatherName: {
    type: String,
    trim: true,
    required: [true, 'Father Name is required'],
  },
  fatherOccupation: {
    type: String,
    trim: true,
    required: [true, 'Father occupation is required'],
  },
  fatherContactNo: {
    type: String,
    required: [true, 'Father Contact No is required'],
  },
  motherName: {
    type: String,
    required: [true, 'Mother Name is required'],
  },
  motherOccupation: {
    type: String,
    required: [true, 'Mother occupation is required'],
  },
  motherContactNo: {
    type: String,
    required: [true, 'Mother Contact No is required'],
  },
});

const localGuardianSchema = new Schema<TLocalGuardian>({
  name: {
    type: String,
    required: [true, 'Name is required'],
  },
  occupation: {
    type: String,
    required: [true, 'Occupation is required'],
  },
  contactNo: {
    type: String,
    required: [true, 'Contact number is required'],
  },
  address: {
    type: String,
    required: [true, 'Address is required'],
  },
});

const studentSchema = new Schema<TStudent>({
  id: {
    type: String,
    required: [true, 'ID is required'],
    unique: true,
  },
  user: {
    type: Schema.Types.ObjectId,
    required: [true, 'User id is required'],
    unique: true,
    ref: 'User',
  },
  name: {
    type: userNameSchema,
    required: [true, 'Name is required'],
  },
  gender: {
    type: String,
    enum: ['male', 'female', 'other'],
    required: [true, 'Gender is required'],
  },
  dateOfBirth: { type: Date },
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
  },
  contactNo: {
    type: String,
    required: [true, 'Contact number is required'],
  },
  emergencyContactNo: {
    type: String,
    required: [true, 'Emergency contact number is required'],
  },
  bloodGroup: {
    type: String,
    enum: ['A+', 'A-', 'B+', 'B-', 'AB+', 'AB-', 'O+', 'O-'],
  },
  presentAddress: {
    type: String,
    required: [true, 'Present address is required'],
  },
  permanentAddress: {
    type: String,
    required: [true, 'Permanent address is required'],
  },
  guardian: {
    type: guardianSchema,
    required: [true, 'Guardian information is required'],
  },
  localGuardian: {
    type: localGuardianSchema,
    required: [true, 'Local guardian information is required'],
  },
  profileImg: { type: String },
  admissionSemester: {
    type: Schema.Types.ObjectId,
    ref: 'AcademicSemester',
  },
  isDeleted: {
    type: Boolean,
    default: false,
  },
  academicDepartment: {
    type: Schema.Types.ObjectId,
    ref: 'AcademicDepartment',
  },
});

export const Student = model<TStudent>('Student', studentSchema);
```
- Zod Validation Schema

Zod is used to validate the incoming data to ensure it meets the required format and constraints.

```javascript
import { z } from 'zod';

const createUserNameValidationSchema = z.object({
  firstName: z.string().min(1).max(20).refine(value => /^[A-Z]/.test(value), {
    message: 'First Name must start with a capital letter',
  }),
  middleName: z.string(),
  lastName: z.string(),
});

const createGuardianValidationSchema = z.object({
  fatherName: z.string(),
  fatherOccupation: z.string(),
  fatherContactNo: z.string(),
  motherName: z.string(),
  motherOccupation: z.string(),
  motherContactNo: z.string(),
});

const createLocalGuardianValidationSchema = z.object({
  name: z.string(),
  occupation: z.string(),
  contactNo: z.string(),
  address: z.string(),
});

export const createStudentValidationSchema = z.object({
  body: z.object({
    password: z.string().max(20),
    student: z.object({
      name: createUserNameValidationSchema,
      gender: z.enum(['male', 'female', 'other']),
      dateOfBirth: z.string().optional(),
      email: z.string().email(),
      contactNo: z.string(),
      emergencyContactNo: z.string(),
      bloodGroup: z.enum(['A+', 'A-', 'B+', 'B-', 'AB+', 'AB-', 'O+', 'O-']),
      presentAddress: z.string(),
      permanentAddress: z.string(),
      guardian: createGuardianValidationSchema,
      localGuardian: createLocalGuardianValidationSchema,
      admissionSemester: z.string(),
      profileImg: z.string(),
      academicDepartment: z.string(),
    }),
  }),
});

const updateUserNameValidationSchema = z.object({
  firstName: z.string().min(1).max(20).optional(),
  middleName: z.string().optional(),
  lastName: z.string().optional(),
});

const updateGuardianValidationSchema = z.object({
  fatherName: z.string().optional(),
  fatherOccupation: z.string().optional(),
  fatherContactNo: z.string().optional(),
  motherName: z.string().optional(),
  motherOccupation: z.string().optional(),
  motherContactNo: z.string().optional(),
});

const updateLocalGuardianValidationSchema = z.object({
  name: z.string().optional(),
  occupation: z.string().optional(),
  contactNo: z.string().optional(),
  address: z.string().optional(),
});

export const updateStudentValidationSchema = z.object({
  body: z.object({
    student: z.object({
      name: updateUserNameValidationSchema,
      gender: z.enum(['male', 'female', 'other']).optional(),
      dateOfBirth: z.string().optional(),
      email: z.string().email().optional(),
      contactNo: z.string().optional(),
      emergencyContactNo: z.string().optional(),
      bloodGroup: z.enum(['A+', 'A-', 'B+', 'B-', 'AB+', 'AB-', 'O+', 'O-']).optional(),
      presentAddress: z.string().optional(),
      permanentAddress: z.string().optional(),
      guardian: updateGuardianValidationSchema.optional(),
      localGuardian: updateLocalGuardianValidationSchema.optional(),
      admissionSemester: z.string().optional(),
      profileImg: z.string().optional(),
      academicDepartment: z.string().optional(),
    }),
  }),
});

export const studentValidations = {
  createStudentValidationSchema,
  updateStudentValidationSchema,
};
```
- Service for Updating Student

Here's where the magic happens. The service will handle the logic for dynamically updating non-primitive fields.

```javascript
import httpStatus from 'http-status';
import mongoose from 'mongoose';
import AppError from '../../errors/AppError';
import { TStudent } from './student.interface';
import { Student } from './student.model';

const updateStudentIntoDB = async (id: string, payload: Partial<TStudent>) => {
  const { name, guardian, localGuardian, ...remainingStudentData } = payload;

  const modifiedUpdatedData: Record<string, unknown> = {
    ...remainingStudentData,
  };

  if (name && Object.keys(name).length) {
    for (const [key, value] of Object.entries(name)) {
      modifiedUpdatedData[`name.${key}`] = value;
    }
  }

  if (guardian && Object.keys(guardian).length) {
    for (const [key, value] of Object.entries(guardian)) {
      modifiedUpdatedData[`guardian.${key}`] = value;
    }
  }

  if (localGuardian && Object.keys(localGuardian).length) {
    for (const [key, value] of Object.entries(localGuardian)) {
      modifiedUpdatedData[`localGuardian.${key}`] = value;
    }
  }

  const result = await Student.findOneAndUpdate({ id }, modifiedUpdatedData, {
    new: true,
    runValidators: true,
  });

  return result;
};

export const StudentServices = {
  updateStudentIntoDB,
};

```
- **Explanation**

  - **Importing Required Modules:** Import necessary files and packages such as http-status, mongoose, AppError, and the Student model.

  - **Handling Non-Primitive Data:** The updateStudentIntoDB function receives an id and a payload as parameters. The id is used to search the document to update, and the payload contains the fields to update. The type for payload is Partial<TStudent> to allow partial updates.

  - **Separating Non-Primitive Fields:** From the payload, we separate the non-primitive fields (name, guardian, localGuardian) and put the remaining fields in the remainingStudentData variable.

  - **Modifying Non-Primitive Data:** 
  
    - **Check for the Presence of Nested Objects:** Verify if the name, guardian, and localGuardian objects are present and contain properties.

    - **Iterate Over Entries:** For each of these objects, iterate over their entries (key-value pairs).

    - **Assign Values Using Template Literals:** Use template literals to dynamically assign the values to their respective keys in the update data. This ensures that nested fields are updated correctly without overwriting existing data.

  - **Updating the Document:** 
  
    - **Perform the Update:** Finally, update the document with the modified data.

    - **Ensure Data Validity:** Set runValidators: true to ensure that all fields, including the updated ones, conform to the defined schema validation rules. This step is crucial for maintaining data integrity and consistency, preventing invalid data from being saved in the database.
  
This approach allows dynamic updates to nested objects without overwriting existing fields, ensuring data integrity and saving bandwidth.

### Conclusion

Updating non-primitive fields in MongoDB documents can be challenging, but with the right approach, it becomes manageable. By dynamically handling nested fields in the backend, you can ensure that updates are efficient and accurate. This method not only preserves existing data but also provides a flexible way to handle partial updates from the frontend.


