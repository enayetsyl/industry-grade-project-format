## Understanding Transactions and Rollbacks in MongoDB

### Introduction

In the world of database management, ensuring data integrity and consistency is crucial. This is where the concepts of transactions and rollbacks come into play. Transactions allow multiple operations to be executed as a single unit of work, ensuring that either all operations succeed or none do. If something goes wrong during the transaction, a rollback reverts the database to its previous state, maintaining data integrity.

- This is the eleventh blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first ten blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide", "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function", "Understanding Populating Referencing Fields in Mongoose" and "Creating a Custom Error Class in an express app". You can check them in the following link.

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

### Benefits of Transactions and Rollbacks

- **Atomicity:** Transactions ensure that all operations within the transaction are completed successfully. If any operation fails, the transaction is aborted, and the database is rolled back to its initial state.
- **Consistency:** Transactions maintain the consistency of the database. The database remains in a valid state before and after the transaction.
- **Isolation:** Transactions are isolated from each other, ensuring that concurrent transactions do not interfere with one another.
- **Durability:** Once a transaction is committed, it remains so, even in the case of a system failure.

### Transactions and Rollbacks in MongoDB

MongoDB supports multi-document transactions, allowing operations on multiple documents and collections to be executed within a single transaction. Here’s an example using the provided code to explain transactions and rollbacks.

### Example: Creating a Student in the Database

Below is a function to create a student in the database using transactions to ensure data integrity.

```javascript
import httpStatus from 'http-status';
import mongoose from 'mongoose';
import config from '../../config';
import AppError from '../../errors/AppError';
import { TStudent } from '../student/student.interface';
import { Student } from '../student/student.model';
import { AcademicSemester } from './../academicSemester/academicSemester.model';
import { TUser } from './user.interface';
import { User } from './user.model';
import { generateStudentId } from './user.utils';

const createStudentIntoDB = async (password: string, payload: TStudent) => {
  // Create a user object
  const userData: Partial<TUser> = {};

  // If password is not given, use the default password
  userData.password = password || (config.default_password as string);

  // Set student role
  userData.role = 'student';

  // Find academic semester info
  const admissionSemester = await AcademicSemester.findById(payload.admissionSemester);

  const session = await mongoose.startSession();

  try {
    session.startTransaction();

    // Set generated ID
    userData.id = await generateStudentId(admissionSemester);

    // Create a user (transaction-1)
    const newUser = await User.create([userData], { session }); // Array

    // Create a student
    if (!newUser.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create user');
    }

    // Set ID and _id as user
    payload.id = newUser[0].id;
    payload.user = newUser[0]._id; // Reference _id

    // Create a student (transaction-2)
    const newStudent = await Student.create([payload], { session });

    if (!newStudent.length) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to create student');
    }

    await session.commitTransaction();
    await session.endSession();

    return newStudent;
  } catch (err) {
    await session.abortTransaction();
    await session.endSession();
    throw new Error('Failed to create student');
  }
};

export const UserServices = {
  createStudentIntoDB,
};
```
### Explanation of the Code

- Start a Session:

```javascript
const session = await mongoose.startSession();
```
- Begin the Transaction:

```javascript
session.startTransaction();
```
- Perform Operations within the Transaction:

  - Generate a unique student ID.
  - Create a new user.
  - Create a new student linked to the user.

- Commit the Transaction if Successful:

```javascript
await session.commitTransaction();
```
- Abort the Transaction in Case of Failure:

```javascript
await session.abortTransaction();
```

### Steps to Implement Transactions and Rollbacks

- Start a MongoDB session:

```javascript
const session = await mongoose.startSession();

```
- Begin the transaction:

```javascript
session.startTransaction();

```
- Perform all database operations within the transaction:

  - Ensure each operation uses the session.
  - Handle errors appropriately.

- Commit the transaction if all operations succeed:

```javascript
await session.commitTransaction();
await session.endSession();
```
- Abort the transaction if any operation fails:

```javascript
await session.abortTransaction();
await session.endSession();
```
By following these steps, you can ensure data consistency and integrity in your applications, leveraging the power of transactions and rollbacks in MongoDB.

### Conclusion

Transactions and rollbacks are powerful tools in database management, providing atomicity, consistency, isolation, and durability (ACID) properties. By implementing transactions in your application, you can ensure that your data remains consistent and your operations are reliable, even in the face of errors. Use the steps outlined above to implement transactions in your MongoDB applications and enhance the robustness of your data operations.