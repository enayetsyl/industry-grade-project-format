## Understanding Populating Referencing Fields in Mongoose

### Introduction 

- In MongoDB and Mongoose, referencing fields allow you to establish relationships between different documents in your database. When you have a reference to another document (or multiple documents), populating those references means that you retrieve and include the actual referenced document(s) instead of just the ObjectId(s).

- This is the ninth blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first eight blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide" and "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function". You can check them in the following link.

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-eslint-and-prettier-1nk6

https://dev.to/md_enayeturrahman_2560e3/folder-structure-in-an-industry-standard-project-271b

https://dev.to/md_enayeturrahman_2560e3/how-to-create-api-in-an-industry-standard-app-44ck

https://dev.to/md_enayeturrahman_2560e3/setting-up-global-error-handler-using-next-function-provided-by-express-96c

https://dev.to/md_enayeturrahman_2560e3/how-to-handle-not-found-route-in-express-app-1d26

https://dev.to/md_enayeturrahman_2560e3/creating-a-custom-send-response-utility-function-in-express-2fg9

https://dev.to/md_enayeturrahman_2560e3/how-to-set-up-routes-in-an-express-app-a-step-by-step-guide-177j

https://dev.to/md_enayeturrahman_2560e3/simplifying-error-handling-in-express-controllers-introducing-catchasync-utility-function-2f3l

### What is Populating Referencing Field?

Populating a referencing field in Mongoose means replacing an ObjectId (or an array of ObjectIds) in a document with the actual document(s) it references. This is particularly useful when you have relationships between different types of data and need to retrieve complete information without making multiple database queries manually.

### Example using Academic Departments

Let's dive into an example using Academic Departments to illustrate how populating referencing fields works:

**Define the Interface for Academic Department**

- Create an interface to define the structure of Academic Department:

```javascript
import { Types } from 'mongoose';

export interface TAcademicDepartment {
  name: string;
  academicFaculty: Types.ObjectId;
}
```

**Define the Schema and Model**

- Define the Mongoose schema for Academic Departments, where academicFaculty is a reference to the AcademicFaculty model:

```javascript
import { Schema, model } from 'mongoose';

const academicDepartmentSchema = new Schema(
  {
    name: {
      type: String,
      required: true,
      unique: true,
    },
    academicFaculty: {
      type: Schema.Types.ObjectId,
      ref: 'AcademicFaculty', // Reference to another model
    },
  },
  {
    timestamps: true,
  }
);

export const AcademicDepartment = model('AcademicDepartment', academicDepartmentSchema);

```

### Service Functions to Retrieve Academic Departments

- Next, implement service functions to interact with Academic Departments in the database:

```javascript
import { AcademicDepartment } from './academicDepartment.model';

const getAllAcademicDepartmentsFromDB = async () => {
  const result = await AcademicDepartment.find().populate('academicFaculty');
  return result;
};

const getSingleAcademicDepartmentFromDB = async (id: string) => {
  const result = await AcademicDepartment.findById(id).populate('academicFaculty');
  return result;
};

export const AcademicDepartmentServices = {
  getAllAcademicDepartmentsFromDB,
  getSingleAcademicDepartmentFromDB,
};

```
**Explanation**

**Populating with .populate():** In Mongoose, the .populate() method allows you to automatically replace specified paths in a document with document(s) from other collection(s) during a query.

**Usage in Service Functions:**

  - **getAllAcademicDepartmentsFromDB:** Retrieves all Academic Departments and populates the academicFaculty field with details from the referenced AcademicFaculty model.

  - **getSingleAcademicDepartmentFromDB:** Retrieves a single Academic Department by its ID and populates the academicFaculty field similarly.

### Benefits of Populating Referencing Fields

  - **Simplified Queries:** Instead of manually fetching related documents, .populate() automates this process, reducing the number of database queries and simplifying code.

  - **Complete Data Retrieval:** Provides a complete view of related data in a single query response, enhancing the efficiency and performance of your application.

### Conclusion

By using .populate() effectively, you can optimize data retrieval and improve the maintainability of your Mongoose-based application, ensuring efficient handling of relationships between documents.
