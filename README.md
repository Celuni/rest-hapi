# Rest Hapi
A Restful API generator built around the Hapi framework and mongoose ODM.

## Features

* Automatic generation of CRUD endpoints with middleware support
* Automatic generation of association endpoints
* Joi validation
* User password encryption support
* Optional token authentication for all generated endpoints
* Swagger docs for all generated endpoints
* Query parameter support for sorting, filtering, pagination, and embedding of associated models

## Installation

```
$ git clone https://github.com/JKHeadley/rest-hapi.git
$ cd rest-hapi
$ npm install
```

## Running the app
```
$ gulp serve:local
```
or just
```
$ gulp
```

## Configuration

Edit the config file relevant to your environment (local, development, production).  The default config
file is ```/api/config.local.js```.  Here you can set the server port, mongodb URI, and authentication.


## Swagger Documentation

Swagger documentation is automatically generated for all endpoints and can be viewed by pointing a browser
at the server URL.  By default this will be ``http://localhost:8124/``.  The swagger docs provide quick 
access to testing your endpoints along with model schema descriptions and query options.

## Creating Endpoints

Restful endpoints are automatically generated based off of any mongoose models that you add to the 
``/api/models`` folder with the file structure of ``{model name}.model.js``.  These models must adhere to the following format:

```javascript
module.exports = function (mongoose) {
    var Schema = new mongoose.Schema({
        ...
        ...fill in schema fields
    });

    Schema.statics = {
        collectionName: /*your model name*/,
        routeOptions: {}
    };

    return Schema;
};
```

As a concrete example, here is a ``user`` model:

``/api/models/user.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "user";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    email: {
      type: Types.String,
      allowNull: false,
      unique: true
    },
    password: {
      type: Types.String,
      allowNull: false,
      required: true,
      exclude: true,
      allowOnUpdate: false
    }
  });
  
  Schema.statics = {
    collectionName: modelName
    routeOptions: {}
  };
  
  return Schema;
};
```

This will generate the following CRUD endpoints:

```
GET /user           Get a list of users
POST /user          Create a new user
GET /user/{_id}     Get a specific user
PUT /user/{_id}     Update a user
DELETE /user/{_id}  Delete a user
```

## Associations

The rest-hapi framework supports model associations that mimic associations in 
a relational database.  This includes one-one, one-many, many-one, and many-many
relationships.  Associations are created by adding the relevant schema fields
and populating the ``associations`` object within ``routeOptions``.  Associations
exists as references to a documents ``_id`` field, and can be populated to return 
the associated object.  See [Querying](#querying) for more details on how to populate
associations.

### ONE_ONE

Below is an example of a one-one relationship between a ``user`` model and a
``dog`` model. Notice the ``dog`` and ``owner`` fields in the schemas.  A schema
field is required for associations of type ``ONE_ONE`` or ``MANY_ONE``.  This
field must match the association name, include a type of ``ObjectId``, and
include a ``ref`` property with the associated model name.

Each association must be added to an ``associations`` object within the
``routeOptions`` object. The ``type`` and ``model`` fields are
required for all associations.

``/api/models/user.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "user";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    email: {
      type: Types.String,
      allowNull: false,
      unique: true
    },
    password: {
      type: Types.String,
      allowNull: false,
      required: true,
      exclude: true,
      allowOnUpdate: false
    },
    dog: {
      type: Types.ObjectId,
      ref: "dog"
    }
  });
  
  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        dog: {
          type: "ONE_ONE",
          model: "dog"
        }
      }
    }
  };
  
  return Schema;
};
```

``/api/models/dog.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "dog";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    name: {
      type: Types.String,
      allowNull: false
    },
    breed: {
      type: Types.String
    },
    owner: {
      type: Types.ObjectId,
      ref: "user"
    }
  });

  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        owner: {
          type: "ONE_ONE",
          model: "user"
        }
      }
    }
  };

  return Schema;
};
```

### ONE_MANY/MANY_ONE

Below is an example of a one-many/many-one relationship between the ``user``
and ``role`` models.  Notice the ``title`` field in the schema.  A schema
field is required for associations of type ``ONE_ONE`` or ``MANY_ONE``.  This
field must match the association name, include a type of ``ObjectId``, and
include a ``ref`` property with the associated model name.

``/api/models/user.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "user";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    email: {
      type: Types.String,
      allowNull: false,
      unique: true
    },
    password: {
      type: Types.String,
      allowNull: false,
      required: true,
      exclude: true,
      allowOnUpdate: false
    },
    title: {
      type: Types.ObjectId,
      ref: "role"
    }

  });
  
  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        title: {
          type: "MANY_ONE",
          model: "role"
        }
      }
    }
  };
  
  return Schema;
};
```

``/api/models/role.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "role";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    name: {
      type: Types.String,
      enum: ["Account", "Admin", "SuperAdmin"],
      allowNull: false
    },
    description: {
      type: Types.String
    }
  });

  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        users: {
          type: "ONE_MANY",
          foreignField: "title",
          model: "user"
        }
      }
    }
  };

  return Schema;
};
```

In this example, a user can belong to one role and a role can be 
assigned to many users.  The ``type`` and ``model`` fields are
required for all associations, and the ``foreignField`` field is 
required for ``ONE_MANY`` type associations.  

Along with the normal CRUD endpoints, the following association 
endpoints will be generated for the ``role`` model:

```
GET /role/{ownerId}/user                Gets all of the users for a role
POST /role/{ownerId}/user               Sets multiple users for a role
PUT /role/{ownerId}/user/{childId}      Add a single user object to a role's list of users
DELETE /role/{ownerId}/user/{childId}   Remove a single user object from a role's list of users
```

### MANY_MANY

Below is an example of a many-many relationship between the ``user`` and
``group`` models. In this relationship a single ``user`` instance can belong
to multiple ``group`` instances and vice versa.

``/api/models/user.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "user";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    email: {
      type: Types.String,
      allowNull: false,
      unique: true
    },
    password: {
      type: Types.String,
      allowNull: false,
      required: true,
      exclude: true,
      allowOnUpdate: false
    }
  });
  
  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        groups: {
          type: "MANY_MANY",
          model: "group"
        }
      }
    }
  };
  
  return Schema;
};
```


``/api/models/group.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "group";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    name: {
      type: Types.String,
      allowNull: false
    },
    description: {
      type: Types.String
    }
  });

  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        users: {
          type: "MANY_MANY",
          model: "user"
        }
      }
    }
  };

  return Schema;
};
```


Along with the normal CRUD endpoints, the following association 
endpoints will be generated for the ``user`` model:

```
GET /user/{ownerId}/group               Gets all of the groups for a user
POST /user/{ownerId}/group              Sets multiple groups for a user
PUT /user/{ownerId}/group/{childId}     Add a single group object to a user's list of groups
DELETE /user/{ownerId}/group/{childId}  Remove a single group object from a user's list of groups
```

and for the ``group`` model:

```
GET /group/{ownerId}/user               Gets all of the users for a group
POST /group/{ownerId}/user              Sets multiple users for a group
PUT /group/{ownerId}/user/{childId}     Add a single user object to a group's list of users
DELETE /group/{ownerId}/user/{childId}  Remove a single user object from a group's list of users
```

#### MANY_MANY linking models

Many-many relationships can include extra fields that contain data specific
to each association instance.  This is accomplished through linking models which
behave similar to pivot tables in a relational database.  Linking model files are
stored in the ``/api/models/linking-models`` directory and follow the same 
``{name}.model.js`` format as normal models.  Below is an example of a many-many
relationship between the ``user`` model and itself through the ``friends`` association.
The extra field ``friendsSince`` could contain a date representing how long the two
associated users have known each other.  This example also displays how models can contain a 
reference to themselves.  


``/api/models/user.model.js``:

```javascript
module.exports = function (mongoose) {
  var modelName = "user";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    email: {
      type: Types.String,
      allowNull: false,
      unique: true
    },
    password: {
      type: Types.String,
      allowNull: false,
      required: true,
      exclude: true,
      allowOnUpdate: false
    }
  });
  
  Schema.statics = {
    collectionName:modelName,
    routeOptions: {
      associations: {
        friends: {
          type: "MANY_MANY",
          model: "user",
          alias: "friend",
          linkingModel: "user_user"
        }
      }
    }
  };
  
  return Schema;
};
```


``/api/models/linking-models/user_user.model.js``:

```javascript
var mongoose = require("mongoose");

module.exports = function () {

  var Types = mongoose.Schema.Types;

  var Model = {
    Schema: {
      friendsSince: {
        type: Types.Date
      }
    },
    modelName: "user_user"
  };

  return Model;
};
```

## Querying

Query parameters can be added to GET requests to filter responses.  These parameters
are structured and function similar to mongoose queries.  Below is a list of currently 
supported parameters:

* $skip
    - The number of records to skip in the database. This is typically used in pagination.

* $limit
    - The maximum number of records to return. This is typically used in pagination.
* $select
    - A list of basic fields to be included in each resource.

* $sort
    - A set of fields to sort by. Including field name indicates it should be sorted ascending, 
    while prepending '-' indicates descending. The default sort direction is 'ascending' 
    (lowest value to highest value). Listing multiple fields prioritizes the sort starting with the first field listed. 

* $embed
    - A set of associations to populate. 

* $where
    - An optional field for raw mongoose queries.

* (field "where" queries)
    - Ex: ``email=test@user.com``
    
Query parameters can either be passed in as a single string, or an array of strings.

### Nested populating
Associations can be populated through the ``$embed`` parameter.  To populate nested associations,
simply chain a parameter with ``.``.  For example, consider the MANY_MANY group-user association
from the example above.  If we populate the users of a group with ``/group?$embed=users`` we might get a 
response like so:

```json
{
    "_id": "58155f1a071468d3bda0fc6e",
    "name": "A-team",
    "users": [
      {
        "user": {
          "_id": "580fc1a0e2d3308609470bc6",
          "email": "test@user.com",
          "title": "580fc1e2e2d3308609470bc8"
        },
        "_id": "58155f6a071468d3bda0fc6f"
      },
      {
        "user": {
          "_id": "5813ad3d0d4e5c822d2f05bd",
          "email": "test2@user.com",
          "title": "580fc1eee2d3308609470bc9"
        },
        "_id": "58155f6a071468d3bda0fc71"
      }
    ]
}
```

However we can further populate each user's ``title`` field with a nested ``$embed``
parameter: ``/group?$embed=users.title`` which could result in the following response:

```json
{
    "_id": "58155f1a071468d3bda0fc6e",
    "name": "A-team",
    "users": [
      {
        "user": {
          "_id": "580fc1a0e2d3308609470bc6",
          "email": "test@user.com",
          "title": {
            "_id": "580fc1e2e2d3308609470bc8",
            "name": "Admin"
          }
        },
        "_id": "58155f6a071468d3bda0fc6f"
      },
      {
        "user": {
          "_id": "5813ad3d0d4e5c822d2f05bd",
          "email": "test2@user.com",
          "title": {
            "_id": "580fc1eee2d3308609470bc9",
            "name": "SuperAdmin"
          }
        },
        "_id": "58155f6a071468d3bda0fc71"
      }
    ]
}
```


## Middleware
Models can support middleware functions for CRUD operations. These
exist under the ``routOptions`` object. Middleware functions must return
 a promise.  The following middleware functions
are available:

* list: 
    - post(request, result, Log)
* find: 
    - post(request, result, Log)
* create:
    - pre(request, Log)
    - post(request, result, Log)
* update: 
    - pre(request, Log)
    - post(request, result, Log)
* delete: 
    - pre(request, Log)
    - post(request, result, Log)


For example, a ``create: pre`` function can be defined to encrypt a users password
using the built-in ``password-helper`` utility.  Notice the use of the ``Q`` library
to return a promise.
 
```javascript
module.exports = function (mongoose) {
  var modelName = "user";
  var Types = mongoose.Schema.Types;
  var Schema = new mongoose.Schema({
    email: {
      type: Types.String,
      allowNull: false,
      unique: true
    },
    password: {
      type: Types.String,
      allowNull: false,
      required: true,
      exclude: true,
      allowOnUpdate: false
    }
  });
  
  Schema.statics= {
    collectionName: modelName
    routeOptions: {
      create: {
        pre: function (request, Log) {
          var deferred = Q.defer();
          var passwordUtility = require('../../api/utilities/password-helper');
          var hashedPassword = passwordUtility.hash_password(request.payload.password);

          request.payload.password = hashedPassword;
          deferred.resolve(request);
          return deferred.promise;
        }
      }
    }
  };
  
  return Schema;
};
```
 
