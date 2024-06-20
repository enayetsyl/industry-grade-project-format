## Updating Non-Primitive Data in an Array Using Transactions and Rollbacks

### Introduction

In this blog, we will explore how to update both primitive and non-primitive data in a MongoDB document using Mongoose. We will specifically focus on updating arrays within documents. Our approach will leverage transactions and rollbacks to ensure data integrity during the update process. We will walk through defining the data types, creating Mongoose schemas, implementing validation with Zod, and finally, updating the data with transaction handling in the service layer.

- This is the thirteenth blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first twelve blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide", "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function", "Understanding Populating Referencing Fields in Mongoose", "Creating a Custom Error Class in an express app", "Understanding Transactions and Rollbacks in MongoDB", "Updating Non-Primitive Data Dynamically in Mongoose", "How to Handle Errors in an Industry-Grade Node.js Application"  and "Creating Query Builders for Mongoose: Searching, Filtering, Sorting, Limiting, Pagination, and Field Selection". You can check them in the following link.

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

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-errors-in-an-industry-grade-nodejs-application-217b

https://dev.to/md_enayeturrahman_2560e3/creating-query-builders-for-mongoose-searching-filtering-sorting-limiting-pagination-and-field-selection-395j

### Defining Data Types

We begin by defining the data types for our course documents

```javascript
import { Types } from 'mongoose';

export type TPreRequisiteCourses = { // Type for PreRequisiteCourses that will be inside an array
  course: Types.ObjectId;
  isDeleted: boolean;
};

// Data structure of our document with one non-primitive and five primitive values
export type TCourse = { 
  title: string;
  prefix: string;
  code: number;
  credits: number;
  isDeleted?: boolean;
  preRequisiteCourses: [TPreRequisiteCourses];
};

```
### Mongoose Schema and Model

Next, we create the Mongoose schemas and models for our course data.

```javascript
import { Schema, model } from 'mongoose';
import {
  TCourse,
  TPreRequisiteCourses,
} from './course.interface';

const preRequisiteCoursesSchema = new Schema<TPreRequisiteCourses>(
  {
    course: {
      type: Schema.Types.ObjectId,
      ref: 'Course',
    },
    isDeleted: {
      type: Boolean,
      default: false,
    },
  },
  {
    _id: false,
  },
);

const courseSchema = new Schema<TCourse>({
  title: {
    type: String,
    unique: true,
    trim: true,
    required: true,
  },
  prefix: {
    type: String,
    trim: true,
    required: true,
  },
  code: {
    type: Number,
    trim: true,
    required: true,
  },
  credits: {
    type: Number,
    trim: true,
    required: true,
  },
  preRequisiteCourses: [preRequisiteCoursesSchema],
  isDeleted: {
    type: Boolean,
    default: false,
  },
});

export const Course = model<TCourse>('Course', courseSchema);
```

### Zod Validation

We use Zod for validation to ensure the data being created or updated adheres to the expected schema.

```javascript
import { z } from 'zod';

const PreRequisiteCourseValidationSchema = z.object({
  course: z.string(),
  isDeleted: z.boolean().optional(),
});

const createCourseValidationSchema = z.object({
  body: z.object({
    title: z.string(),
    prefix: z.string(),
    code: z.number(),
    credits: z.number(),
    preRequisiteCourses: z.array(PreRequisiteCourseValidationSchema).optional(),
    isDeleted: z.boolean().optional(),
  }),
});

const updatePreRequisiteCourseValidationSchema = z.object({
  course: z.string(),
  isDeleted: z.boolean().optional(),
});

const updateCourseValidationSchema = z.object({
  body: z.object({
    title: z.string().optional(),
    prefix: z.string().optional(),
    code: z.number().optional(),
    credits: z.number().optional(),
    preRequisiteCourses: z
      .array(updatePreRequisiteCourseValidationSchema)
      .optional(),
    isDeleted: z.boolean().optional(),
  }),
});

export const CourseValidations = {
  createCourseValidationSchema,
  updateCourseValidationSchema
};

```

- Handling Validation for Updates

The issue we should focus on is that for createCourseValidationSchema, all fields but one are required. If we use it for updating by using partial (because not all fields require an update), it will not work. The required fields will not become optional by using the partial method. So, I created a new validation schema for the update and made all fields optional. This way, from the front end, users can update any field.

### Service Layer

I will skip the content of the route and controller files and directly move to the service file

```javascript
import httpStatus from 'http-status';
import mongoose from 'mongoose';
import AppError from '../../errors/AppError';
import { TCourse } from './course.interface';
import { Course } from './course.model';

const updateCourseIntoDB = async (id: string, payload: Partial<TCourse>) => {
  const { preRequisiteCourses, ...courseRemainingData } = payload; // Separate primitive and non-primitive data

  const session = await mongoose.startSession(); // Initiate session for transaction

  try {
    session.startTransaction();  // Start transaction

    // Step 1: Update primitive course info
    const updatedBasicCourseInfo = await Course.findByIdAndUpdate(
      id,
      courseRemainingData, // Pass primitive data
      {
        new: true,
        runValidators: true, // Run validators
        session, // Pass session
      },
    );

    // Throw error if update fails
    if (!updatedBasicCourseInfo) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course');
    }

    // Check if there are any prerequisite courses to update
    if (preRequisiteCourses && preRequisiteCourses.length > 0) {
      // Filter out the deleted fields
      const deletedPreRequisites = preRequisiteCourses
        .filter((el) => el.course && el.isDeleted) // Fields with isDeleted property value true
        .map((el) => el.course); // Only take id for deletion

      const deletedPreRequisiteCourses = await Course.findByIdAndUpdate(
        id,
        {
          $pull: { // Use $pull operator to remove objects with matching ids in deletedPreRequisites
            preRequisiteCourses: { course: { $in: deletedPreRequisites } },
          },
        },
        {
          new: true,
          runValidators: true, // Run validators
          session, // Pass session
        },
      );

      // Throw error if deletion fails
      if (!deletedPreRequisiteCourses) {
        throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course');
      }

      // Filter out courses that need to be added (isDeleted property value is false)
      const newPreRequisites = preRequisiteCourses?.filter(
        (el) => el.course && !el.isDeleted,
      );

      // Perform write operation to update newly added fields
      const newPreRequisiteCourses = await Course.findByIdAndUpdate(
        id,
        {
          $addToSet: { preRequisiteCourses: { $each: newPreRequisites } },
        }, // Use $addToSet operator to avoid duplication
        {
          new: true,
          runValidators: true,
          session,
        },
      );

      if (!newPreRequisiteCourses) {
        throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course');
      }
    }

    // Commit and end session for successful operation
    await session.commitTransaction();
    await session.endSession();

    // Perform find operation to return the data
    const result = await Course.findById(id).populate(
      'preRequisiteCourses.course',
    );

    return result;
  } catch (err) {
    console.log(err);
    await session.abortTransaction(); // Abort transaction on error
    await session.endSession(); // End session
    throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course'); // Throw error
  }
};

export const CourseServices = {
  updateCourseIntoDB
};

```

### Conclusion

In this blog, we have walked through the process of updating primitive and non-primitive data in a MongoDB document using Mongoose. By utilizing transactions and rollbacks, we ensure the integrity of our data during complex update operations. This approach allows for robust handling of both primitive and non-primitive updates, making our application more resilient and reliable











/*----------------------------------





Draft














*/---------------------------------------










## Updating non-primitive data in an array.

- In this blog we will learn how to update primitive and non-primitive data using transaction and rollback. Our focus will be on how to update an array. 

- We will begin with the type of the data in the document.

```javascript
import { Types } from 'mongoose';

export type TPreRequisiteCourses = { //This is the type for PreRequisiteCourses that will hold inside an array
  course: Types.ObjectId;
  isDeleted: boolean;
};

// this is the data structure of our document. It has one non-primitive and five primitive values.
export type TCourse = { 
  title: string;
  prefix: string;
  code: number;
  credits: number;
  isDeleted?: boolean;
  preRequisiteCourses: [TPreRequisiteCourses];
};
```
- Now we will see the mongoose schema and model for the data.

```javascript
import { Schema, model } from 'mongoose';
import {
  TCourse,
  TPreRequisiteCourses,
} from './course.interface';

const preRequisiteCoursesSchema = new Schema<TPreRequisiteCourses>(
  {
    course: {
      type: Schema.Types.ObjectId,
      ref: 'Course',
    },
    isDeleted: {
      type: Boolean,
      default: false,
    },
  },
  {
    _id: false,
  },
);

const courseSchema = new Schema<TCourse>({
  title: {
    type: String,
    unique: true,
    trim: true,
    required: true,
  },
  prefix: {
    type: String,
    trim: true,
    required: true,
  },
  code: {
    type: Number,
    trim: true,
    required: true,
  },
  credits: {
    type: Number,
    trim: true,
    required: true,
  },
  preRequisiteCourses: [preRequisiteCoursesSchema],
  isDeleted: {
    type: Boolean,
    default: false,
  },
});

export const Course = model<TCourse>('Course', courseSchema);
```

- Next we will see the zod validation for the data.

```javascript
import { z } from 'zod';

const PreRequisiteCourseValidationSchema = z.object({
  course: z.string(),
  isDeleted: z.boolean().optional(),
});

const createCourseValidationSchema = z.object({
  body: z.object({
    title: z.string(),
    prefix: z.string(),
    code: z.number(),
    credits: z.number(),
    preRequisiteCourses: z.array(PreRequisiteCourseValidationSchema).optional(),
    isDeleted: z.boolean().optional(),
  }),
});

const updatePreRequisiteCourseValidationSchema = z.object({
  course: z.string(),
  isDeleted: z.boolean().optional(),
});

const updateCourseValidationSchema = z.object({
  body: z.object({
    title: z.string().optional(),
    prefix: z.string().optional(),
    code: z.number().optional(),
    credits: z.number().optional(),
    preRequisiteCourses: z
      .array(updatePreRequisiteCourseValidationSchema)
      .optional(),
    isDeleted: z.boolean().optional(),
  }),
});

export const CourseValidations = {
  createCourseValidationSchema,
  updateCourseValidationSchema
};
```
- The issue where we should focus is that for createCourseValidationSchema all fields but one is required. If we use it for updating by using partial (because not all fields require update) it will not work. The required field will not become option by using partial method. So i created new validation schema for the update and make all fields optional. So from the front-end user can update any field.

- I will skip the content of  route and controller file and directly move to service file. 

```javascript
import httpStatus from 'http-status';
import mongoose from 'mongoose';
import AppError from '../../errors/AppError';
import { TCourse, TCoursefaculty } from './course.interface';
import { Course, CourseFaculty } from './course.model';

const updateCourseIntoDB = async (id: string, payload: Partial<TCourse>) => { // getting id and req.body from the controller as params
  const { preRequisiteCourses, ...courseRemainingData } = payload; // from the payload separating the primitive and non-primitive data

  const session = await mongoose.startSession(); // There will be three write operation so transaction is used so handle any adverse situation.

  try {
    session.startTransaction();  // Starting the session

    //step1: primitive course info update
    const updatedBasicCourseInfo = await Course.findByIdAndUpdate(
      id,
      courseRemainingData, // Passing primitive data
      {
        new: true,
        runValidators: true, // Running validator
        session, // Passing session
      },
    );

// throwing error if could not update 
    if (!updatedBasicCourseInfo) {
      throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course');
    }

    // check if there is any pre requisite courses to update
    if (preRequisiteCourses && preRequisiteCourses.length > 0) {
      // filter out the deleted fields
      const deletedPreRequisites = preRequisiteCourses
        .filter((el) => el.course && el.isDeleted) // fields that has isDeleted property value true
        .map((el) => el.course); // only taking id that is required for delete.

      const deletedPreRequisiteCourses = await Course.findByIdAndUpdate(
        id,
        {
          $pull: { // using $pull operator from the course array removing the objects that's id match with the id stored in the deletedPreRequisites
            preRequisiteCourses: { course: { $in: deletedPreRequisites } },
          },
        },
        {
          new: true,
        runValidators: true, // Running validator
        session, // Passing session
        },
      );
// throwing error if could not delete 
      if (!deletedPreRequisiteCourses) {
        throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course');
      }

      // filter out the course that need to add. Separating items that's isDeleted property value is false.
      const newPreRequisites = preRequisiteCourses?.filter(
        (el) => el.course && !el.isDeleted,
      );

    // Now performing another write operation to update newly added fields
      const newPreRequisiteCourses = await Course.findByIdAndUpdate(
        id,
        {
          $addToSet: { preRequisiteCourses: { $each: newPreRequisites } },
        }, // SaddToSet operator is used to enter items once avoiding duplication
        {
          new: true,
          runValidators: true,
          session,
        },
      );

      if (!newPreRequisiteCourses) {
        throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course');
      }
    }
// Committing and ending session for successful operation
    await session.commitTransaction();
    await session.endSession();

// Find operation is done to return the data
    const result = await Course.findById(id).populate(
      'preRequisiteCourses.course',
    );

    return result;
  } catch (err) {
    console.log(err);
    await session.abortTransaction(); // Aborting transaction for any error
    await session.endSession(); //Ending the session 
    throw new AppError(httpStatus.BAD_REQUEST, 'Failed to update course'); // Throwing the error
  }
};

export const CourseServices = {
  updateCourseIntoDB
};
```