## Creating Query Builders for Mongoose: Searching, Filtering, Sorting, Limiting, Pagination, and Field Selection

In this blog, we will explore how to implement searching, filtering, sorting, limiting, pagination, and field selection in isolation. Afterward, we will create a query builder component that combines all these functionalities, making them reusable across different models. Let's dive in

- This is the thirteenth blog of my series where I am writing how to write code for an industry-grade project so that you can manage and scale the project.  

- The first twelve blogs of the series were about "How to set up eslint and prettier in an express and typescript project", "Folder structure in an industry-standard project", "How to create API in an industry-standard app", "Setting up global error handler using next function provided by express", "How to handle not found route in express app", "Creating a Custom Send Response Utility Function in Express", "How to Set Up Routes in an Express App: A Step-by-Step Guide", "Simplifying Error Handling in Express Controllers: Introducing catchAsync Utility Function", "Understanding Populating Referencing Fields in Mongoose", "Creating a Custom Error Class in an express app", "Understanding Transactions and Rollbacks in MongoDB", "Updating Non-Primitive Data Dynamically in Mongoose" and "How to Handle Errors in an Industry-Grade Node.js Application". You can check them in the following link.

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


### Introduction

Efficiently querying a database is crucial for optimizing application performance and enhancing user experience. Mongoose, a popular ODM (Object Data Modeling) library for MongoDB and Node.js, provides a powerful way to interact with MongoDB. By creating a query builder class, we can streamline the process of constructing complex queries, making our code more maintainable and scalable.

### Searching and Filtering

In an HTTP request, we can send data in three ways: through the body (large chunks of data), params (dynamic data like an ID), and query (fields needed for querying). Query parameters, which come as an object provided by the Express framework, consist of key-value pairs. Let's start with a request containing two queries:

```javascript
/api/v1/students?searchTerm=chitta&email=enayet@gmail.com
```

Here, searchTerm is used for searching with a partial match, while email is used for filtering with an exact match. The searchable fields are defined on the backend.

### Method Chaining

Understanding method chaining is crucial for this implementation. If you're unfamiliar with it, you can read my blog on Method Chaining in Mongoose: A Brief Overview.

https://dev.to/md_enayeturrahman_2560e3/method-chaining-in-mongoose-a-brief-overview-44lm

### Basic query

We will apply our learning in the following code:

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async () => {
  const result = await Student.find()
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB,
};

```
- The code for search:

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => { // received query as a param from the controller file. We do not know the type of the query as it could be anything, so we set it as a record; its property will be a string and value is unknown.

  let searchTerm = '';   // SET DEFAULT VALUE. If no query is sent from the frontend, it will be an empty string.

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // fields in the document where the search will take place. We should keep it in a separate constant file. You can add or remove more fields as per your requirement.

  // IF searchTerm IS GIVEN, SET IT
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  const searchQuery = Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  });

  // Here we are chaining the query above and executing it below using await

  const result = await searchQuery
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB,
};

```
- We can use method chaining to make the above code cleaner as follows

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => { // received query as a param from the controller file. We do not know the type of the query as it could be anything, so we set it as a record; its property will be a string and value is unknown.

  let searchTerm = '';   // SET DEFAULT VALUE. If no query is sent from the frontend, it will be an empty string.

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // fields in the document where the search will take place. We should keep it in a separate constant file. You can add or remove more fields as per your requirement.

  // IF searchTerm IS GIVEN, SET IT
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  // The find operation is performed in the Student Collection. The MongoDB $or operator is used here. The studentSearchableFields array is mapped, and for each item in the array, the property in the DB is searched with the search term using regex to get a partial match. 'i' is used to make the search case-insensitive.
  const result = await Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  })
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB,
};

```
- Now we will implement filtering. Here we will match the exact value: 

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  const searchQuery = Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  });

  // FILTERING FUNCTIONALITY:
  
  const excludeFields = ['searchTerm'];
  excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

  // explained earlier
  const result = await searchQuery
    .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB,
};
```
- For sorting, the query will be as follows

```javascript
/api/v1/students?sort=email //for ascending
/api/v1/students?sort=-email //for descending
```
- The code for sorting will be as follows, including previous queries:

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  const searchQuery = Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  });

  // FILTERING FUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort'];
  excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

  // explained earlier
  const filteredQuery = searchQuery // change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
    .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  let sort = '-createdAt';  // By default, sorting will be based on the createdAt field in descending order, meaning the last item will be shown first. 

  if (query.sort) {
    sort = query.sort as string; // if the query object has a sort property, then its value is assigned to the sort variable. 
  }

  const sortQuery = await filteredQuery.sort(sort); // method chaining is done on filteredQuery

  return sortQuery;
};

export const StudentServices = {
  getAllStudentsFromDB,
};
```

- Now we will limit data using the query, and it will be done on top of the above code:

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  const searchQuery = Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  });

  // FILTERING FUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort', 'limit'];
  excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

  // explained earlier
  const filteredQuery = searchQuery // change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
    .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  let sort = '-createdAt';  // By default, sorting will be based on the createdAt field in descending order, meaning the last item will be shown first. 

  if (query.sort) {
    sort = query.sort as string; // if the query object has a sort property, then its value is assigned to the sort variable. 
  }

  const sortedQuery = filteredQuery.sort(sort); // change the variable name to sortedQuery and await is removed from it. so here we are chaining on filteredQuery

  let limit = 0;  // if no limit is given, all data will be shown

  if (query.limit) {
    limit = parseInt(query.limit as string);  // if limit is given, then its value is assigned to the limit variable. Since the value will be a string, it is converted into an integer.
  }

  const limitedQuery = await sortedQuery.limit(limit); // method chaining is done on filteredQuery

  return limitedQuery;
};

export const StudentServices = {
  getAllStudentsFromDB,
};
```

- Let's apply pagination:

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  const searchQuery = Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  });

  // FILTERING FUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort', 'limit', 'page'];
  excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

  // explained earlier
  const filteredQuery = searchQuery // change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
    .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  let sort = '-createdAt';  // By default, sorting will be based on the createdAt field in descending order, meaning the last item will be shown first. 

  if (query.sort) {
    sort = query.sort as string; // if the query object has a sort property, then its value is assigned to the sort variable. 
  }

  const sortedQuery = filteredQuery.sort(sort); // change the variable name to sortedQuery and await is removed from it. so here we are chaining on filteredQuery

  let limit = 0;  // if no limit is given, all data will be shown

  if (query.limit) {
    limit = parseInt(query.limit as string);  // if limit is given, then its value is assigned to the limit variable. Since the value will be a string, it is converted into an integer.
  }

  const limitedQuery = sortedQuery.limit(limit); // change the variable name to limitedQuery and await is removed from it. so here we are chaining on sortedQuery

  let page = 1;  // if no page number is given, by default, we will go to the first page.

  if (query.page) {
    page = parseInt(query.page as string);  // if the page number is given, then its value is assigned to the page variable. Since the value will be a string, it is converted into an integer.
  }

  const skip = (page - 1) * limit;  // Suppose we are on page 1. To get the next set of documents, we need to skip the first 10 docs if the limit is 10. On page 2, we need to skip the first 20 docs. So the page number is multiplied with the limit. 

  const paginatedQuery = await limitedQuery.skip(skip);  // method chaining is done on limitedQuery
  
  return paginatedQuery;
};

export const StudentServices = {
  getAllStudentsFromDB,
};
```

- Finally, we will limit the data field as follows:

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress']; // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

  const searchQuery = Student.find({
    $or: studentSearchableFields.map((field) => ({
      [field]: { $regex: searchTerm, $options: 'i' },
    })),
  });

  // FILTERING FUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort', 'limit', 'page', 'fields'];
  excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

  // explained earlier
  const filteredQuery = searchQuery // change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
    .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  let sort = '-createdAt';  // By default, sorting will be based on the createdAt field in descending order, meaning the last item will be shown first. 

  if (query.sort) {
    sort = query.sort as string; // if the query object has a sort property, then its value is assigned to the sort variable. 
  }

  const sortedQuery = filteredQuery.sort(sort); // change the variable name to sortedQuery and await is removed from it. so here we are chaining on filteredQuery

  let limit = 0;  // if no limit is given, all data will be shown

  if (query.limit) {
    limit = parseInt(query.limit as string);  // if limit is given, then its value is assigned to the limit variable. Since the value will be a string, it is converted into an integer.
  }

  const limitedQuery = sortedQuery.limit(limit); // change the variable name to limitedQuery and await is removed from it. so here we are chaining on sortedQuery

  let page = 1;  // if no page number is given, by default, we will go to the first page.

  if (query.page) {
    page = parseInt(query.page as string);  // if the page number is given, then its value is assigned to the page variable. Since the value will be a string, it is converted into an integer.
  }

  const skip = (page - 1) * limit;  // Suppose we are on page 1. To get the next set of documents, we need to skip the first 10 docs if the limit is 10. On page 2, we need to skip the first 20 docs. So the page number is multiplied with the limit. 

  const paginatedQuery = limitedQuery.skip(skip);  // change the variable name to paginatedQuery and await is removed from it. so here we are chaining on limitedQuery

  let fields = ''; // if no fields are given, by default, all fields will be shown.

  if (query.fields) {
    fields = query.fields as string; // if the query object has fields, then its value is assigned to the fields variable.
  }

  const selectedFieldsQuery = await paginatedQuery.select(fields);  // method chaining is done on limitedQuery
  
  return selectedFieldsQuery;
};

export const StudentServices = {
  getAllStudentsFromDB,
};

```
### Query builder class

- Currently, all the queries apply to the Student model. If we want to apply them to a different model, we would have to rewrite them, which violates the DRY (Don't Repeat Yourself) principle. To avoid repetition, we can create a class where all the queries are available as methods. This way, whenever we need to apply these queries to a new collection, we can simply create a new instance of that class. This approach will enhance scalability and maintainability and make the codebase cleaner.

```javascript
import { FilterQuery, Query } from 'mongoose';  // Import FilterQuery and Query types from mongoose.

class QueryBuilder<T> { // Declare a class that will take a generic type
  public modelQuery: Query<T[], T>; // Property for model. The query is run on a model, so we named it modelQuery. You can name it anything else. After the query, we receive an array or object, so its type is set as an object or an array of objects.
  public query: Record<string, unknown>; // The query that will be sent from the frontend. We do not know what the type of query will be, so we kept its property as a string and its value as unknown. 

  // Define the constructor
  constructor(modelQuery: Query<T[], T>, query: Record<string, unknown>) {
    this.modelQuery = modelQuery;
    this.query = query;
  }

  search(searchableFields: string[]) { // Method for the search query, taking searchableFields array as a parameter.
    const searchTerm = this?.query?.searchTerm; // Take the search term from the query using this.

    if (searchTerm) { // If search term is available in the query, access the model using this.modelQuery and perform the search operation.
      this.modelQuery = this.modelQuery.find({
        $or: searchableFields.map(
          (field) =>
            ({
              [field]: { $regex: searchTerm, $options: 'i' },
            }) as FilterQuery<T>,
        ),
      });
    }

    return this; // Return this for method chaining in later methods. 
  }

  filter() { // Method for filter query without any parameter. The query is performed on this.modelQuery using method chaining and then returns this.
    const queryObj = { ...this.query }; // Copy the query object

    // Filtering
    const excludeFields = ['searchTerm', 'sort', 'limit', 'page', 'fields'];
    excludeFields.forEach((el) => delete queryObj[el]);

    this.modelQuery = this.modelQuery.find(queryObj as FilterQuery<T>);
    return this;
  }

  sort() { // Method for sort query without any parameter. The query is performed on this.modelQuery using method chaining and then returns this. Also, the sort variable is adjusted so now sorting can be done based on multiple fields.
    const sort = (this?.query?.sort as string)?.split(',')?.join(' ') || '-createdAt';
    this.modelQuery = this.modelQuery.sort(sort as string);
    return this;
  }

  paginate() { // Method for paginate query without any parameter. The query is performed on this.modelQuery using method chaining and then returns this.
    const page = Number(this?.query?.page) || 1;
    const limit = Number(this?.query?.limit) || 10;
    const skip = (page - 1) * limit;

    this.modelQuery = this.modelQuery.skip(skip).limit(limit);
    return this;
  }

  fields() { // Method for fields query without any parameter. The query is performed on this.modelQuery using method chaining and then returns this.
    const fields = (this?.query?.fields as string)?.split(',')?.join(' ') || '-__v';
    this.modelQuery = this.modelQuery.select(fields);
    return this;
  }
}

export default QueryBuilder;

```

- How can we apply the QueryBuilder to any model? We will see an example for the Student model, but in the same way, it can be applied to any model.

```javascript
import QueryBuilder from '../../builder/QueryBuilder';
import { studentSearchableFields } from './student.constant';  // Import studentSearchableFields from a separate file.

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {
  const studentQuery = new QueryBuilder( // Create a new instance of the QueryBuilder class.
    Student.find() // This will act as a modelQuery inside the class.
      .populate('admissionSemester')
      .populate({
        path: 'academicDepartment',
        populate: {
          path: 'academicFaculty',
        },
      }),
    query, // This will act as a query inside the class.
  )
    .search(studentSearchableFields) // Method chaining on studentQuery.
    .filter() // Method chaining on studentQuery.
    .sort() // Method chaining on studentQuery.
    .paginate() // Method chaining on studentQuery.
    .fields(); // Method chaining on studentQuery.

  const result = await studentQuery.modelQuery; // Perform the final asynchronous operation on studentQuery.
  return result;
};

export const StudentServices = {
  getAllStudentsFromDB,
};

```
### Conclusion 

This comprehensive code snippet handles search, filtering, sorting, pagination, and field selection in a MongoDB query using Mongoose. It processes the incoming query object and constructs a MongoDB query with appropriate modifications and chaining of methods for each operation.









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

```javascript

```

```javascript

```

```javascript

```

















































## Query builders for searching, filtering, sorting, limiting, pagination and field limiting.

- In this blog we will learn how to do searching, filtering, sorting, limiting, pagination and field limiting in isolation. Then we will create query builder component by combining all so that they can be reused. 

### Searching and filtering

- In a http request we can send data in three ways through body(large chank of data), params(dynamic data e.g. id) and query (fields that need to query). query is an object that is provided by the express framework which consist data in key value pair.

- Lets look a request with two quaries

```javascript
/api/v1/students?searchTerm=chitta&email=enayet@gmail.com
```

here searchTerm is for searching something. In this case we will search using "chitta" value. The fields(searchable fields. e.g. name.firstname, email, presentAddress) where search will happen is not mentioned here. It will be defined at the backend. Here we will match partially not exactly. on the other hand email will be used for filtering data so we know the field and it will be exact match not partial.

- In order to implement this properly you need to understand method chaining. If you do not have idea about it you can read my following blog
https://dev.to/md_enayeturrahman_2560e3/method-chaining-in-mongoose-a-brief-overview-44lm

- We will apply our learning in the following code

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async () => {
  const result = await Student.find()
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- below will be the code for search

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => { // received query as a param from the controller file. We do not the type of the query as it could be anything. So set it as a record its property will be a string and value is unknown.

  let searchTerm = '';   // SET DEFAULT VALUE. So no query is sent from the frontend then it will be empty string

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // fields in the document where search will take place. We should keep it in a separate constant file. You can add or less more field as par your requirement.

  // IF searchTerm  IS GIVEN SET IT
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

const searchQuery = Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });

// Here we are chaining the query above and executing it in below using await

  const result = await searchQuery
  .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- We can use method chaining to make the above code more cleaner as follows

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => { // received query as a param from the controller file. We do not the type of the query as it could be anything. So set it as a record its property will be a string and value is unknown.

  let searchTerm = '';   // SET DEFAULT VALUE. So no query is sent from the frontend then it will be empty string

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // fields in hte document where search will take place. We should keep it in a separate constant file. You can add or less more field as par your requirement.

  // IF searchTerm  IS GIVEN SET IT
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

// Below find operation is performed in the Student Collection. mongodb $or operator is used here. studentSearchableFields array is mapped and for each item in the array in the db the property is searched with the search term using regex to get partial match. 'i' is used to make the search case insensitive.
  const result = await Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- now we will implement filtering. Here we will match exact value. 

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

const searchQuery = Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });

// FILTERING fUNCTIONALITY:
  
  const excludeFields = ['searchTerm'];
   excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

// explained earlier
   const result = await searchQuery
  .find()
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- for sorting the query will be as follows

```javascript
/api/v1/students?sort=email //for ascending
/api/v1/students?sort=-email //for descending
```

- the code for sorting will be as follows including previous queries

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

const searchQuery = Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });

// FILTERING fUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort'];
   excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

// explained earlier
   const filteredQuery =  searchQuery //change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
  .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

    let sort = '-createdAt'  // By default sorting will be based on createAt field at descending order means last item will be shown first. 

    if(query.sort){
      sort = query.sort as string; // if query object has sort property then its value is assigned to sort variable. 
    }

    const sortQuery = await filterQuery.sort(sort) // method chaining is done on filterQuery and 

  return sortQuery;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- now we will limit data using query. and it will be done on top of above code .

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

const searchQuery = Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });

// FILTERING fUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort', 'limit'];
   excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

// explained earlier
   const filteredQuery =  searchQuery //change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
  .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

    let sort = '-createdAt'  // By default sorting will be based on createAt field at descending order means last item will be shown first. 

    if(query.sort){
      sort = query.sort as string; // if query object has sort property then its value is assigned to sort variable. 
    }

    const sortQuery =  filterQuery.sort(sort) // method chaining is done on filterQuery and 

    const limit = 1
    if (query.limit){
      limit = Number(query.limit);
    }

  const limitQuery = await sortQuery.limit(limit)

  return limitQuery;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- key changes are limit is added in the excluded field array.

- await is removed from the sortQuery

- new variable limit is declared with the value of 1.

- using if block it is checked that whether there is any limit property in the query. if yes then it is assign to the limit variable by converting it to number. 

- method chaining is applied to the sortQuery and at last return the limitQuery.

- now we will learn pagination. for pagination we send two query with the request. first one is the page number and second one is the data in each page. another property skip will be used at the backend to perform the task. the formula for searching data will be as follows

limit = limit from frontend(e.g. 10), page = page from frontend(e.g. 2), skip = (page -1)* limit. the full code will be as follows

```javascript
import { Student } from './student.model';

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

const searchQuery = Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });

// FILTERING fUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort', 'limit', 'page'];
   excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

// explained earlier
   const filteredQuery =  searchQuery //change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
  .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

    let sort = '-createdAt'  // By default sorting will be based on createAt field at descending order means last item will be shown first. 

    if(query.sort){
      sort = query.sort as string; // if query object has sort property then its value is assigned to sort variable. 
    }

    const sortQuery =  filterQuery.sort(sort) // method chaining is done on filterQuery and 

   // PAGINATION FUNCTIONALITY:

   let page = 1; // SET DEFAULT VALUE FOR PAGE 
   let limit = 1; // SET DEFAULT VALUE FOR LIMIT 
   let skip = 0; // SET DEFAULT VALUE FOR SKIP


  // IF limit IS GIVEN SET IT
  
  if (query.limit) {
    limit = Number(query.limit);
  }

  // IF page IS GIVEN SET IT

  if (query.page) {
    page = Number(query.page);
    skip = (page - 1) * limit;
  }

  const paginateQuery = sortQuery.skip(skip);

  const limitQuery = await paginateQuery.limit(limit);


  return limitQuery;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```

- the changes are as follows
- in the excludeFields array i added page. 
- default value for page and limit variable is set to 1 and for skip value is set to 0. 
- if query has limit then it is converted to number and assigned to limit. same thing done for page. 
- (page-1) * limit this formula is assigned to the skip variable. method chaining is applied to sortQuery to get paginateQuery. method chaining is applied to paginateQuery to get limitQuery.

- at last return the limitQuery

- now we will learn how to field limiting using query.

- for field limiting data will come from front end as follows

```javascript
/api/v1/students?fields=-name(name property will be removed from the result)
/api/v1/students?fields=name,age(Only name and age property will be send as a result.)
```
- in the above case if we console log the query we can see following result

```javascript
{query: {fields: 'name,email'}}
```
- the name of the fields that we will send as a query will be an object and its value will be string and each property will be seperated by comma. Our code will be as follows for field filtering

```javascript
const getAllStudentsFromDB = async (query: Record<string, unknown>) => {  // explained earlier
  const queryObj = { ...query }; // copying req.query object so that we can mutate the copy object 

  let searchTerm = '';   // explained earlier

  const studentSearchableFields = ['email', 'name.firstName', 'presentAddress'] // explained earlier

  // explained earlier
  if (query?.searchTerm) {
    searchTerm = query?.searchTerm as string; 
  }

const searchQuery = Student.find({
     $or: studentSearchableFields.map((field) => ({
       [field]: { $regex: searchTerm, $options: 'i' },
    })),
   });

// FILTERING fUNCTIONALITY:
  
  const excludeFields = ['searchTerm', 'sort', 'limit', 'page', 'fields'];
   excludeFields.forEach((el) => delete queryObj[el]);  // DELETING THE FIELDS SO THAT IT CAN'T MATCH OR FILTER EXACTLY

// explained earlier
   const filteredQuery =  searchQuery //change the variable name to filteredQuery and await is removed from it. so here we are chaining on searchQuery
  .find(queryObj)
    .populate('admissionSemester')
    .populate({
      path: 'academicDepartment',
      populate: {
        path: 'academicFaculty',
      },
    });

    let sort = '-createdAt'  // By default sorting will be based on createAt field at descending order means last item will be shown first. 

    if(query.sort){
      sort = query.sort as string; // if query object has sort property then its value is assigned to sort variable. 
    }

    const sortQuery =  filterQuery.sort(sort) // method chaining is done on filterQuery and 

   // PAGINATION FUNCTIONALITY:

   let page = 1; // SET DEFAULT VALUE FOR PAGE 
   let limit = 1; // SET DEFAULT VALUE FOR LIMIT 
   let skip = 0; // SET DEFAULT VALUE FOR SKIP


  // IF limit IS GIVEN SET IT
  
  if (query.limit) {
    limit = Number(query.limit);
  }

  // IF page IS GIVEN SET IT

  if (query.page) {
    page = Number(query.page);
    skip = (page - 1) * limit;
  }

  const paginateQuery = sortQuery.skip(skip);

  const limitQuery =  paginateQuery.limit(limit);

let fields = '-__v'; // SET DEFAULT VALUE

  if (query.fields) {
    fields = (query.fields as string).split(',').join(' ');

  }
 const fieldQuery = await limitQuery.select(fields);

  return fieldQuery;

};

export const StudentServices = {
  getAllStudentsFromDB
};
```
- the changes are as follows

- fields word added to the excludeFields array.
- default value for fields is set -__v. so it will be excluded when data will be returned to the frontend.
- if query has fields property then the value of its will be split based on comma and again join using a space. method chaining is applied on limit query to get fieldQuery that is returned to the front end

- now above all queries applies on student model. if we want to apply them to a different model then we have to rewrite them again, so we cannot maintain DRY concept. in order to avoid repetition we can create a class where all the query will be available as a method so that whenever we need to apply them in a new collection we will just create a new instance of that class. this will increase scalability and maintainability and make code base more cleaner.

```javascript
import { FilterQuery, Query } from 'mongoose';  //Imported FilterQuery and Query type from mongoose.

class QueryBuilder<T> { // Declare a class that will take generic type
  public modelQuery: Query<T[], T>; // This is the property for model. As the query is run on a model so we named it modelQuery. you can name it anything else. After query we receive an array or object so its type set as an object or an array of object.
  public query: Record<string, unknown>; // This is the query that will be sent from the frontend. We do not know what will be the type for query so kept its property as string and value unknown. 

// define the construction
  constructor(modelQuery: Query<T[], T>, query: Record<string, unknown>) {
    this.modelQuery = modelQuery;
    this.query = query;
  }

  search(searchableFields: string[]) { // Method is created for search query and passed searchableFields array as a param.
    const searchTerm = this?.query?.searchTerm; //we are taking search term from the query using this.

    if (searchTerm) { // if search term is available in the query we are accessing the model using this.modelQuery and performing search operation that was described earlier.
      this.modelQuery = this.modelQuery.find({
        $or: searchableFields.map(
          (field) =>
            ({
              [field]: { $regex: searchTerm, $options: 'i' },
            }) as FilterQuery<T>,
        ),
      });
    }

    return this; // this contains modelQuery and query so returning this will help to method chaining in the later methods. 
  }

// for filter query we created a method called filter that does not take any param. everything is same for filter query that we explained earlier. the main difference is that the query is performed on this.modelQuery using method chaining and then return the this.
  filter() {
    const queryObj = { ...this.query }; // copy

    // Filtering
    const excludeFields = ['searchTerm', 'sort', 'limit', 'page', 'fields'];

    excludeFields.forEach((el) => delete queryObj[el]);

    this.modelQuery = this.modelQuery.find(queryObj as FilterQuery<T>);

    return this;
  }

// for sort query we created a method called sort that does not take any param. everything is same for sort query that we explained earlier. the main difference is that the query is performed on this.modelQuery using method chaining and then return the this. also sort variable is adjusted so now sort can be done based on multiple fields. 
  sort() {
    const sort =
      (this?.query?.sort as string)?.split(',')?.join(' ') || '-createdAt';
    this.modelQuery = this.modelQuery.sort(sort as string);

    return this;
  }

// for paginate query we created a method called paginate that does not take any param. everything is same for paginate query that we explained earlier. the main difference is that the query is performed on this.modelQuery using method chaining and then return the this.
  paginate() {
    const page = Number(this?.query?.page) || 1;
    const limit = Number(this?.query?.limit) || 10;
    const skip = (page - 1) * limit;

    this.modelQuery = this.modelQuery.skip(skip).limit(limit);

    return this;
  }

// for fields query we created a method called fields that does not take any param. everything is same for fields query that we explained earlier. the main difference is that the query is performed on this.modelQuery using method chaining and then return the this.
  fields() {
    const fields =
      (this?.query?.fields as string)?.split(',')?.join(' ') || '-__v';

    this.modelQuery = this.modelQuery.select(fields);
    return this;
  }
}

export default QueryBuilder;
```
- How can we apply the QueryBuilder on any model? We will see example for Student model but in the same way it can be applied to any model.

```javascript
import QueryBuilder from '../../builder/QueryBuilder';
import { studentSearchableFields } from './student.constant';  // studentSearchableFields kept in a separate file and imported from there

const getAllStudentsFromDB = async (query: Record<string, unknown>) => {
  
  const studentQuery = new QueryBuilder( // New instance is called for the QueryBuilder class
    Student.find()// this will act as a modelQuery inside the class
      .populate('admissionSemester')
      .populate({
        path: 'academicDepartment',
        populate: {
          path: 'academicFaculty',
        },
      }),
    query, // this will act as a query inside the class
  )
    .search(studentSearchableFields) // method chaining on studentQuery
    .filter() // method chaining on studentQuery
    .sort() // method chaining on studentQuery
    .paginate() // method chaining on studentQuery
    .paginate() // method chaining on studentQuery
    .fields(); // method chaining on studentQuery

  const result = await studentQuery.modelQuery; // final asynchronous operation is done on studentQuery.

  return result;
};

export const StudentServices = {
  getAllStudentsFromDB
};
```