---
layout: post
title:  "Type Juggling and MySQL: A Dangerous Combination"
date:   2020-03-15 18:08:32 -0300
categories: security
---
Type juggling is when you trick the application into processing something as a different type than intended by the programmers. This is usually a problem on languages with dynamic typing. Those languages usually have some sort of type coercion in place and if you try to compare two variables of different types one of them will usually be implicitly converted to the other type and then the comparison will be done.

For example, that allows the comparison `1 == "1"` in PHP to evaluate to true. That can however be exploited in malicious ways.

Let's make a PHP script that's vulnerable to better understand the problem:

```php
<?php
$server_secret = "verysecretstring";

$params = json_decode($_GET['params'], true);

if ($params['secret'] != $server_secret) {
        echo "Wrong secret";
        exit;
}

echo "Correct secret";
```

In this script the `$params` variable receives a deserialized json string, then we check the value of the `secret` parameter against the server secret, if it's the same string then the rest of the request is processed, if not we return an error.

Normally an user will send the secret as a string inside the json, and that will work as intended, but what happens if we change the type to something else? How will the server deal with this? Let's try it out and see what happens:

```json
{
  "secret": true
}
```

If we try that we'll get `Correct secret` as the output.

This happens because in PHP  `(boolean)"anystring"` evaluates as true. Before doing the comparison the string is casted to boolean so it ends up as `true == true` which is a truthy statement. So in this case you can use type juggling to bypass the authentication. One way to avoid this is to use the === and !== operators for comparisons, this will check the type as well as the value and no implicit casting will be done. If we change the operator in our test script and send the same json that allowed the type juggle before we'll get `Wrong secret` as the output.

One thing most people don't realize is that you can exploit this down the stack as well. Let's create a table in MySQL called users, this table has an api_token column which is a varchar column:

```sql
CREATE TABLE Users (
id int UNSIGNED AUTO_INCREMENT PRIMARY KEY,
api_token VARCHAR(30) NOT NULL,
index(api_token)
);
```

The api_token column is used to authenticate an user, every time someone makes a request we send the user supplied token to the application which then queries the database and authenticates the user if there is a match.

Let's insert an user:

```sql
INSERT INTO Users (api_token) VALUES ('5d458fc4641360a8b46b2226507252');
```

What happens if we query this table passing something that's not a string to compare to api_token?

```sql
SELECT * FROM Users where api_token = 5
```

This will yield the following result:

```
+----+--------------------------------+
| id | api_token                      |
+----+--------------------------------+
|  1 | 5d458fc4641360a8b46b2226507252 |
+----+--------------------------------+
1 row in set, 1 warning (0.00 sec)
```

This happens because the api_token column ends up being casted to integer, this cast will end up resulting in a numeric 5 because this string starts with an 5:

```sql
SELECT CAST("5d458fc4641360a8b46b2226507252" AS int);
```

```
+--------------------------------------------------+
| CAST("5d458fc4641360a8b46b2226507252" AS SIGNED) |
+--------------------------------------------------+
|                                                5 |
+--------------------------------------------------+
```

This is why you should always double check your application when you do serialization of user input. Even if your application is not exploitable to type juggling issues something down your stack such as the database may be exploitable and the damage could be significant, in this example an unauthorized user could end up authenticating as a user without knowing the api_token.
