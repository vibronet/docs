---
title: Configure Auth0 to Use a Database as an Identity Provider
description: Learn how to configure Auth0 to use your database as an identity provider.
crews: crew-2
toc: true
---
# Configure Auth0 to Use a Database as an Identity Provider

In this article, we will show you how to configure Auth0 to use your database as an identity provider. You will need to:

1. Create an Auth0 database connection
2. Create database action scripts
3. Add configuration parameters

## Step 1: Create an Auth0 database connection

The first thing you will do is create a database connection in Auth0. To do so, follow these steps:

1. In the Dashboard, navigate to [Connections > Database](${manage_url}/#/connections/database).

2. Click **+ Create DB Connection**.

3. Configure the connection's **settings** as requested.

| **Parameter** | **Definition** |
| - | - |
| **Name** | The name of the connection. The name must start and end with an alphanumeric character, contain only alphanumeric characters and dashes, and not exceed 35 characters. |
| **Requires Username** | Forces users to provide a username *and* email address during registration. |
| **Username length** | Sets the minimum and maximum length for a username. |
| **Disable Sign Ups** | Prevents sign ups to your application. You will still be able to create users with your API credentials or via the Dashboard, however. |

4. Click **Create** to proceed.

![Database connections](/media/articles/connections/database/database-connections.png)

Once Auth0 creates your connection, you'll have the following tabs (in addition to the **Settings** tab):

  * Password Policy
  * Custom Database
  * Applications
  * Try Connection

4. Switch over to the **Custom Database** tab.

5. Toggle the **Use my own database** switch to enable this feature.

![Custom database tab](/media/articles/connections/database/custom-database.png)

## Step 2: Create database action scripts

Toggling the **Use my own database** switch enables the **Database Action Scripts** area. This is where you will create scripts to configure how authentication works when using your database.

You **must** configure a login script; additional scripts for user functionality, such as password resets, are optional.

You can write your own database action scripts or you can begin by selecting a template from the **Templates** dropdown and modifying it as necessary.

The available database actions are as follows.

Name | Description | Parameters
-------|-------------|-----------
Login <br/><span class="label label-danger">Required</span> | Executes each time a user attempts to log in. | `email`, `password`
Create | Executes when a user signs up. | `user.email`, `user.password`
Verify | Executes after a user follows the verification link. | `email`
Change Password | Executes when a user clicks on the confirmation link after a reset password request. | `email`, `newPassword`
Get User | Retrieves a user profile from your database without authenticating the user. | `email`
Delete | Executes when a user is deleted from the API or Auth0 dashboard. | `id`

::: note
When creating users, Auth0 calls the **Get User** script before the **Create** script. Be sure to implement both database action scripts if you are creating new users.
:::

### Create the Login script

The Login script will run each time a user attempts to log in. You can write your own Login script or select a template from the **Templates** dropdown.

::: note
If you are using [IBM's DB2](https://www.ibm.com/analytics/us/en/technology/db2/) product, [click here](/connections/database/db2-script) for a sample login script.
:::

![Database action script templates](/media/articles/connections/database/mysql/db-connection-login-script.png)

For example, the MySQL Login template:

```js
function login(email, password, callback) {
  var connection = mysql({
    host: 'localhost',
    user: 'me',
    password: 'secret',
    database: 'mydb'
  });

  connection.connect();

  var query = "SELECT id, nickname, email, password " +
    "FROM users WHERE email = ?";

  connection.query(query, [email], function (err, results) {
    if (err) return callback(err);
    if (results.length === 0) return callback(new WrongUsernameOrPasswordError(email));
    var user = results[0];

    bcrypt.compare(password, user.password, function (err, isValid) {
      if (err) {
        callback(err);
      } else if (!isValid) {
        callback(new WrongUsernameOrPasswordError(email));
      } else {
        callback(null, {
          id: user.id.toString(),
          nickname: user.nickname,
          email: user.email
        });
      }
    });

  });
}

```

The above script connects to a MySQL database and executes a query to retrieve the first user with `email == user.email`. With the `bcrypt.compareSync` method, it then validates that the passwords match, and if successful, returns an object containing the user profile information including `id`, `nickname`, and `email`. This script assumes that you have a `users` table containing these columns. Note that `id` returned by Login script is used to construct `user_id` attribute of user profile. If you are using multiple custom database connections then value of `id` must be unique across all the custom database connections to avoid and `user_id` collisions. Our recommendation is to prefix the value of `id` with connection name without any whitespace in the resulting value.

## Step 3: Add configuration parameters

You can store parameters, like the credentials required to connect to your database, in the Settings section below the script editor. These will be available to all of your scripts and you can access them using the global configuration object.

You can access parameter values using the `configuration` object in your database action scripts, for example: `configuration.MYSQL_PASSWORD`.

![Custom database settings](/media/articles/connections/database/mysql/db-connection-configurate.png)

Use the added parameters in your scripts to configure the connection. For example, in the MySQL Login template:

```js
function login (username, password, callback) {
  var connection = mysql({
    host     : configuration.MYSQL_HOST,
    user     : 'me',
    password : configuration.MYSQL_PASSWORD,
    database : 'mydb'
  });
}
```