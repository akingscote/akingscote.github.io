---
title: "Role Based Authenticated RESTful API - Python & Flask"
date: "2017-07-03"
categories: 
  - "development"
  - "python"
  - "security"
  - "web-development"
---

This post shows how to create an authenticated RESTful API using Python and Flask. The basic idea is you can customise what HTTP operations a user group can perform. For example, a user with the admin role could perform everything, whereas a read-only user could only perform HTTP GET operations. The full example code can be found [here](https://github.com/akingscote/authenticated_RESTful_API_flask_sqlite)
A database has been created that has four tables:
- Accounts - Fake Data that our users want to access
- User - user account info
- Role - a table containing unique roles each with a primary key
- UserRoles - a table managing the relationship between each user and their role

In this example, i have created "admin", "developer", "standard" roles. The idea is that any users with the "admin" role will have access to everything, users with the "developer" role will have access to the accounts information and user with the "standard" role will have front end access only.

Using flask-restless, two API's are created. One publishing the "User" data and one publishing the "Accounts" data.

```
# only admin can see user api
MANAGER.create_api(User, methods = ['GET', 'POST', 'PATCH', 'DELETE'], preprocessors=user_api_settings)
```

# admin and developer can see Accounts API MANAGER.create_api(Accounts, methods = ['GET', 'POST', 'PATCH', 'DELETE'], preprocessors=account_api_settings)

Each API defines which table they are getting data from. The table Object Relational Mapping (ORM) schema is imported from the create_database file. The methods indicate which HTTP verbs can be performed on the API. RESTful API's are nearly always built around HTTP as it readily conforms to the standards as described in [Roy Fieldings dissertation.](https://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf) There are two additional variables that can be supplied which are **preprocessors** and **postprocessors**. Preprocessors can be used to apply a function to any request parameters/message before the request is processed, whereas postprocessors allow functions to be applied to the response data. In this case, before a request is completed we want to check that a user is allowed access to the API so therefore we need to use the preprocessors.

```
account_api_settings = dict(POST = [account_auth], GET_SINGLE = [account_auth], GET_MANY = [account_auth], PATCH_SINGLE = [account_auth], PATCH_MANY = [account_auth], DELETE_SINGLE = [account_auth], DELETE_MANY = [account_auth])
```
The preprocessor for the user table calls this user_api_settings dictionary. Here, more specific HTTP operations can be restricted. In this example, im not too bothered about what specific operations a certain type of user can before, but rather whether or not they can access the API.

```
def account_auth(*args, **kwargs):
    con = engine.connect()
    role = con.execute("SELECT name FROM role WHERE id = (SELECT role_id FROM user_roles WHERE user_id == {})".format(current_user.id))
    role = role.first()[0]
    con.close()
    if not role in ("developer", "admin"):
        raise ProcessingException(description='Not authenticated!', code=401)
```

The account_auth function will be called whenever one of the previously defined operations are attempted (PATCH_SINGE, DELETE_MANY etc..). The function uses the SQLAlchemy engine to connect to the database with incredible speed. The SQL query looks kind of confusing because its basically two queries in one statement. Using the current_user.id value, we can get the unique primary key of the currently logged in user. If nobody is logged in, the value will be none and the query will not work, which will raise an authentication message. The first query looks like this: `SELECT role_id FROM user_roles WHERE user_id == {})".format(current_user.id)` Which simple gets the role_id for the logged in user. Remember, the user_roles table is where the mappings from user to roles is stored. From this, we can then use this returned value to perform another lookup to find out which role name the returned role_id corresponds too. As there will only be one role as they are unique, we can call the .first() function on the results to turn the results into a list. Fetching the first element in the list will always return our role (unless the current_user.id is None). If the role returned does not match either developer, or admin then the user will not be able to log in.

By authenticating the API like this, we create a completely customizable authentication which can be fine tuned down to the individual roles. This solution is create for small applications, but for larger solutions the role permissions should be defined in the database to reduce code.

Below is a video of the web application showing that the admin user (with the role admin) can access both the User and Accounts API endpoint, whereas the developer user (with the developer role) can only access the Accounts API endpoint and not the user endpoint. The video shows if nobody is logged in, then neither API can be accessed.

Check out the video demonstration [here](/assets/video/authenticated_restful_api_flask_python.mp4)

