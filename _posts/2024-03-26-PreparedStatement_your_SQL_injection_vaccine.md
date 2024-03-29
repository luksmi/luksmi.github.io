---
title: 'PreparedStatement: your SQL injection vaccine'
date: 2024-03-26 20:37:12 +0100
categories: [guide, integration, security   ]
tags: [programming, jdbc, sql, java, security, databases]
mermaid: true
image:
  path: /assets/images/posts/2024-03-26/img.jpg
  lqip: /assets/images/posts/2024-03-26/img.jpg
pin: true
---

In this article I'm going to show you the key difference between `Statement` and `PreparedStatement` classes from JDBC - and how the latter protects us from SQL injection attacks.
To prove it, we're going to run a PostgreSQL database and tweak its configuration to see every executed SQL query. Also, we'll integrate the database with two simple Java programs: one being vulnerable and the other secure.

This is a beginner-level step-by-step guide. You can follow along the commands presented in this article - they are ready to be copied and pasted right into your command line.

## Prerequisites
To make this tutorial accessible for everyone, I tried to keep required toolkit as minimal as possible: 
- **Java SDK**\
  How to install: my recommendation is to use **SDKMAN!** (install it here: [https://sdkman.io/install](https://sdkman.io/install){:target="_blank"}) which can seamlessly manage various JDK versions on your computer. Then run `sdk install java 21.ea.35-open` and verify the JDK installation by executing `java -version` 
- **Docker Engine**\
  How to install: go to [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/){:target="_blank"} and follow instructions relevant to your OS
- **Git** (optional)\
Source code used in this article is available on my github ([https://github.com/luksmi/jdbc-demo](https://github.com/luksmi/jdbc-demo){:target="_blank"}) and can be easily cloned. Instead, you can copy and paste code snippets directly to an IDE or files on your machine.\
How to install: [https://git-scm.com/book/en/v2/Getting-Started-Installing-Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git){:target="_blank"}


## What is SQL injection?
{: id="whatsinjection"}
Surprisingly, as of 2024, it still appears on the recent OWASP Top Ten list of vulnerabilities. It's fairly simple: the root cause is constructing SQL queries using user-provided input without proper sanitization. If a user is smart enough, the entered values can change the semantics of the query. 

Let's illustrate this with a classic example: consider an application with some form of authentication logic. This logic is based on a SELECT query that searches the database for any users matching provided username and password:


```java
String query = "SELECT * FROM users WHERE username = '"
                + username + "' AND password = '" + password + "'"; 
```

A user is considered authenticated if any row is returned. Otherwise, the login attempt gets rejected.

Imagine an attacker that enters `' OR 1=1--` as username and `abc` (or anything else) as password. If you join strings from the snippet above, the final query sent to the database server would then be:
```sql
SELECT * FROM users WHERE username = '' OR 1=1--' AND password = 'abc'
```
The condition `1=1` always evaluates to true `true`, and `--` is a comment symbol that causes the database to ignore the rest of the query. This means that all rows from the users table are returned, and the attacker ends up logged in as a random user.

It's not the only scenario in which injecting malicious SQL code can lead to unpredictable results. Other security issues include unauthorized access, manipulation, or even deletion of data. We certainly don't want this to happen.


## What is JDBC?

JDBC is a basic connectivity layer between a Java application and the underlying database. Every JPA implementation builds on top of JDBC, including famous Hibernate. 

Simply speaking: if your Java application interacts with any relational DB, then, most likely, JDBC is involved along the way. The JDBC API exposes two basic classes representing SQL queries: `Statement` and `PreparedStatement`. The third one is `CallableStatement` designed for stored procedures and we will leave that for now.

JDBC is shipped as a part of standard JDK installation, so you don't have to add any external dependencies. Apart from the database-specific driver that we're going to download later.

## Setup

### Downloading and running PostgreSQL

> Note: at the time of writing this article, the most recent version of PostgreSQL is `16.2`. If you need a different one, check what's avaliable on [https://hub.docker.com/\_/postgres](https://hub.docker.com/_/postgres){:target="_blank"} and replace `16.2` with appropriate number. However, I don't guarantee that the instructions that I provide here will work with other releases.
{: .prompt-info }

To have DBMS up and running: 

- Pull PostgreSQL Docker image in version `16:2` :
```shell
docker pull postgres:16.2
```

- The command below will use freshly pulled image to run a container named `postgres-demo`. Inside the container, there's going to be the database server we need:
```shell
docker run -d -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -p 5432:5432 --name postgres-demo postgres:16.2
```

- Then, log into this container:
```shell
docker exec -it postgres-demo /bin/bash
```
- Inside the container, connect to the database using `psql` client. Pass credentials previously specified in the `docker run` command:
```shell
psql -h localhost -p 5432 -U postgres -W
```
When prompted, type `postgres` as password.


### Inserting sample data

In order to show how users' data can be protected against (or exposed to) SQL injection attacks, we need... the data. Let's stick to the log in scenario described earlier.

- Create new database:
```sql
CREATE DATABASE jdbc_demo;
```
- And then connect:
```
\connect jdbc_demo;
```
When prompted for password, type `postgres`.

- Create a simple table named `users` that wil store username and password of registered users:
```sql
CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        username VARCHAR(255) NOT NULL UNIQUE,
        password VARCHAR(255) NOT NULL
);
```

- Then insert some users into the table:
```sql
INSERT INTO users VALUES ('1', 'Olaf', '1806'); 
INSERT INTO users VALUES ('2', 'Donald', '1918'); 
INSERT INTO users VALUES ('3', 'Emmanuel', '1789');
```
- We can validate they're stored happily by executing:
```sql
SELECT * FROM users;
```
and the output should be:
```
 id | username | password
----+----------+----------
    1 | Olaf     | 1806
    2 | Donald   | 1918
    3 | Emmanuel | 1789
(3 rows)
```
- To perform configuration steps, exit `psql` client:
```
\q
```
and then exit `postgres-demo` container:
```
exit
```

### PostgresSQL configuration

To make *pg* log every query, we need to edit `postgresql.conf` file:
- Copy `postgresql.conf` file from inside the container to your local machine by executing:
```shell
docker cp postgres-demo:/var/lib/postgresql/data/postgresql.conf postgresql.conf
```
This command will create `postgresql.conf` in the current working directory.

Then:
- Open `postgresql.conf` file with a text editor. We're particularly interested in two entries: `log_statement` and `log_min_messages`. You should see they're commented out with a `#` prefix (at least in `16.2` version)
- Find `log_min_messages` setting, uncomment it by removing `#` and set its value to `notice` 
- Find `log_statement` setting, uncomment it by removing `#` and set its value to `'all'` (pay attention to single quotes here)

- Copy edited file back to the container to overwrite the original one:
```shell
docker cp postgresql.conf postgres-demo:/var/lib/postgresql/data/postgresql.conf
```
- Restart the container:
```shell
docker restart postgres-demo
```
- Follow the logs coming produced by the container:
```shell
docker logs -f postgres-demo
```
The last line you should see is:
```
2024-03-24 19:16:16.151 UTC [1] LOG:  database system is ready to accept connections
```

Keep this terminal window open as here we'll be able to see every SQL query executed by the DB server.

### Compiling the source code

- Fire up the second terminal window. We're going to need two of them: the first one for monitoring DB logs, and the second one for compiling and running code examples.
- Execute the command below to clone `jdbc-demo` repository from my github:  
```shell
git clone git@github.com:luksmi/jdbc-demo.git
```
> Note: thise repository contains two simple Java programs: `Vulnerable.java` and `Secure.java`. These files are exactly the same as presented in this article. If you prefer, you can copy them directly from the article instead of cloning the repo.

- Navigate to the repo:
```shell
cd jdbc-demo
```

- Compile `Vulnerable.java` and `Secure.java` files:
```shell
javac Vulnerable.java Secure.java
```
- There are going to be `.class` files created in your working directory. Let's package them into `.jar` files:
```shell
jar cf Vulnerable.jar Vulnerable.class && jar cf Secure.jar Secure.class
```
Boom, executable `Vulnerable.jar` and `Secure.jar` appeared. We'll come back to that later. 

- As the last step of this setup go to [https://jdbc.postgresql.org/download/](https://jdbc.postgresql.org/download/){:target="_blank"} and download JDBC driver for PostgreSQL. Save downloaded `.jar` driver to `jdbc-demo` directory. At the moment of writing this post, the file name is `postgresql-42.7.3.jar`, but the version will most probably change in the future.
	

> Note: If you're working with an IDE, instead of downloading the jar driver, paste the following into your `pom.xml` file:
>
>```xml
><groupId>org.postgresql</groupId>
>    <artifactId>postgresql</artifactId>
>    <version>42.7.3</version>
></dependency>
>``` 
{: .prompt-info }



## Vulnerable example

> **Disclaimer**: for simplicity, programs below have some flaws. In a real-world project these flaws should be addressed before deploying to production:
> - database credentials should be kept encrypted on a config server
> - database should store salted hashes of users' passwords
> - instead of throwing `SQLException` you should use a try-with-resources statement, so that the connection and any resources are closed properly in case of errors
> - consider some connection pooling library instead of managing connections manually
>
{: .prompt-info }

Let's finally see some code.

The following snippet shows a program susceptible to SQL injection, with a poorly designed authentication mechanism. Intentionally. It accepts two command line arguments: `username` and `password` and looks for users in the database matching provided credentials. If any user is found, the program logs `Logged in as: <username>`, otherwise `Username or password is incorrect` is printed. 

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class Vulnerable {

    public static void main(String[] args) throws SQLException {
        if (args.length != 2) {
            System.out.println("Please provide username and password only");
            return;
        }
        String username = args[0];
        String password = args[1];

        Connection connection = DriverManager.getConnection(
                "jdbc:postgresql://localhost:5432/jdbc_demo",
                "postgres",
                "postgres"
        );

        String query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'";
        Statement statement = connection.createStatement();

        ResultSet resultSet = statement.executeQuery(query);
        if (resultSet.next()) {
            String fetchedUsername = resultSet.getString("username");
            System.out.println("Logged in as: " + fetchedUsername);
        } else {
            System.out.println("Username or password is incorrect");
        }

        resultSet.close();
        statement.close();
        connection.close();
    }
}
```
{: file='Vulnerable.java'}

The program uses exactly the same query construction pattern as described in the previous ["What is SQL injection?"](#whatsinjection) paragraph. Let's see it in action:

### Running Vulnerable example
> In this tutorial, we’re going to explicitly include `postgresql-42.7.3.jar` dependency with every program run. In a real-world project we'd use some kind of a build tool (Maven or Gradle) to package both our code and the external dependencies into a single fat jar. Again, note that the `42.7.3` version could change since this article was poasted.
{: .prompt-info }

- Suppose that a guy named Donald has registered an account in our app some time ago and now he's trying to log in. Provides `Donald` as the username (first command line argument) and `1918` as his password (second argument):
```shell
java -cp Vulnerable.jar:postgresql-42.7.3.jar Vulnerable "Donald" "1918"
```
output:
```
Logged in as: Donald
```
Provided credentials match those stored in the database for a user with a username `Donald` . From now on, the person is authenticated successfully. Everything went smooth as expected.

- What if Donald forgot his password?
```shell
java -cp Vulnerable.jar:postgresql-42.7.3.jar Vulnerable "Donald" "forgot"
```
output:
```
Username or password is incorrect
```
So far so good.
- Now imagine Donald is seeking security holes in our app. Tries all the standard tricks - one of them being SQL injection. Enters `' OR 1=1--` and `anything`:
```shell
java -cp Vulnerable.jar:postgresql-42.7.3.jar Vulnerable "' OR 1=1--" "anything"
```
output:
```
Logged in as: Olaf
```

Oops! For some reason Donald managed to authenticate as Olaf. Without knowing his password, or even username! Now Donald, using Olaf's account, has all the privileges and roles assigned to this account. What if Olaf was a user with admin priviliges? This could lead to a great harm if Donald (the attacker) had bad intentions.

### Explanation

Take a look at database logs:
```sql
SELECT * FROM users WHERE username = '' OR 1=1--' AND password = 'anything'
```
The part after `--` is ignored, so the effective query will search for a record in `users` table that meets one of the two conditions:
1. `username = ''` 
2. `1=1` 

Obviously `1` is always equal to `1`, So the condition is met for every record in the table. That's why all records are returned, and `Vulnerable.java` is happy with the first one - in this case, the `Olaf` record.

As mentioned earlier, the problematic line is:

```java
String query = "SELECT * FROM users WHERE username = '"
                + username + "' AND password = '" + password + "'"; 
```

By gluing together the following strings:
- `SELECT * FROM users WHERE username = '` 
- `' OR 1=1--` 
- `' AND password = '`
- `anything` 
- `'` 

we get the malicious SQL statement that we have just seen.

## Secure example

Below is a slightly refactored version of the previous program:
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class Secure {

    public static void main(String[] args) throws SQLException {
        if (args.length != 2) {
            System.out.println("Please provide username and password only");
            return;
        }
        String username = args[0];
        String password = args[1];

        Connection connection = DriverManager.getConnection(
                "jdbc:postgresql://localhost:5432/jdbc_demo",
                "postgres",
                "postgres"
        );

        String queryTemplate = "SELECT * FROM users WHERE username = ? AND password = ?";
        PreparedStatement preparedStatement = connection.prepareStatement(queryTemplate);
        preparedStatement.setString(1, username);
        preparedStatement.setString(2, password);

        ResultSet resultSet = preparedStatement.executeQuery();
        if (resultSet.next()) {
            String fetchedUsername = resultSet.getString("username");
            System.out.println("Logged in as: " + fetchedUsername);
        } else {
            System.out.println("Username or password is incorrect");
        }

        resultSet.close();
        preparedStatement.close();
        connection.close();
    }
}
```
{: file='Secure.java'}

The only difference between this program and the vulnerable one lies between lines 23 and 26. Firstly, we create a string which is just an SQL query template. The `?` signs in the template indicate that actual values will be supplied later. Then we invoke `prepareStatement` method on a `Connection` instance and pass it the query template. The `prepareStatement` method returns a `PreparedStatement` instance. 

The actual values replacing the `?` placeholders in the `PreparedStatement` object are specified on lines 25 and 26.

### Running Secure example
- First, let's test the standard positive path...
```shell
java -cp Secure.jar:postgresql-42.7.3.jar Secure "Donald" "1918"
```
output:
```
Logged in as: Donald
```

- ...and the negative one:
```shell
java -cp Secure.jar:postgresql-42.7.3.jar Secure "Donald" "forgot"
```
output:
```
Username or password is incorrect
```

No regression. Beautiful. Looking at the first terminal window you can already spot that the database engine prints out two log entries per execution - instead of only one as when running `Vulnerable.java`. But...
- ...have we fixed the SQL injection vulnerability?
```shell
java -cp Secure.jar:postgresql-42.7.3.jar Secure "' OR 1=1--" "anything"
```
output:
```
Username or password is incorrect
```

Seems like we did. The exact same arguments that did the trick in `Vulnerable.java` example don't work for `Secure.java`. Database logs tell us the following:
```
2024-03-25 22:05:10.489 UTC [73] LOG:  execute <unnamed>: SELECT * FROM users WHERE username = $1 AND password = $2
2024-03-25 22:05:10.489 UTC [73] DETAIL:  parameters: $1 = ''' OR 1=1--', $2 = 'anything'
```

It clearly says that the query template: `SELECT * FROM users WHERE username = $1 AND password = $2` was processed separately from its parameters: `$1 = ''' OR 1=1--'` and `$2 = 'anything'`.

### Explanation

Here's what happens step-by-step:
- When creating a `PreparedStatement` the database prepares this query for execution understanding that there's a parameter `?` whose value will be supplied later.
- When calling `preparedStatement.setString(1, username)` the JDBC driver sends value of username to the database as data to be used in the prepared query (not as part of the SQL command itself)
- Malicious input `' OR 1=1--` is treated as a literal string. The reason why the are three single quotes at the beginning of `''' OR 1=1--'` is because SQL standard surrounds string values with single quotes. This is the first occurence. The other two are because the single quote provided as part of the malicious input needs to be escaped with... another single quote. Thanks to that SQL knows that `'` starts a string and `''` means literally `'`.


## Summary
To wrap it up, the application is considered vulnerable to SQL injection if user input is directly concatenated with the database query, changing its semantics. Results are dramatically different than those intended by the author of the software. 

To mitigate this risk, we can use the `PreparedStatement` class that accepts a query template - in contrary to the `Statement` class, which operates on the final raw query. `PreparedStatement` processes the query template and its parameters separately, as seen in the database logs. Remember: even if we're not dealing hands-on with JDBC on a daily basis, the ORM frameworks we rely on are built of top of it. 

BTW, speaking of all "the ORM frameworks"... is anyone here who's using an ORM framework other than Hibernate? ;) 

