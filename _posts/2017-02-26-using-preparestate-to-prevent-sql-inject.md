--- 
title: Using PrepareState to prevent SQL Inject
layout: post
---

In this article you will understand how SQL injection attacks are used to compromise the security of a web application, and how to write code more securely to protect against this type of attack.

Let's start with a SQL injection example. Suppose there's an online application with a login page. Registered users of the application can do the specific business. Tom is a legitimate registered user of this system. And he tries to login to the application with the following credenticals:
* `Username: tom@abc.com`
* `Password: tom123`


![login page](/images/posts/login_page.png) 

Unfortunately, the password `tom123` does not seem to work for Tom's account. Before trying to click the `Forgot username/password` button, Tom tries entering the same password followed by a single qute `'`character. Something like:
* `Username: tom@abc.com`
* `Password: tom123'`

However, something broke. The application gets to crash with a `HTTP 500 Internal Server Error`. So what's wrong with the system? Why adding a single quote to the password caused the application to crash?

In order to find the root cause why the error occurred, let's first give the application's authentication method for the login page. 

    String email = request.getParameter("email");
    String password = request.getParameter("password");
    
    String sql = "select * from users where (email =" + email + " and password =" + password + ")";
    
    Connection connection = pool.getConnection();
    Statement statement = connection.createState();
    ResultSet result = statement.executeQuery(sql);
    
    if (result.next()) {
        loggedIn = true;
        # Successfully logged in and redirect to user profile page
    } else {
        # Auth failure - Redirect to Login Page
    }

Putting Tom's email and password into the sql query statement, it becomes `select * from users where email = 'tom@abc.com' and password = 'tom123''`. When the query is processed, the last quote which is interpreted by the database server as a string delimiter does not have a closing/matching `'` character, which causes a `SQL syntax error`, resulting in the `HTTP 500 Internal Server Error`.

At this point we know that injecting characters interpreted by the database server is known a `SQL Inject`. However, it's not just `'` characters that can be injected, entire strings can be injected too. What if this could be used to alter the pupose of the SQL statement entirely? Let's try entering the following credentials:

* `Username: tom@abc.com`
* `Password: ' or 1=1)#`

It works, Tom succeed to login the system which redirect to the profile page! So what's going on?

Note in MySQL the `#` character is used for code comments (For SQL Server, it's single line comment is `--`). So that's to say, in the above sql query statement everything to the right of the `#` charachter is commented out, including the extra `'` and `)` character. The real sql query is `select * from users where (email = 'tom@abc.com' and password = ' ' or 1=1)` which the condition is true and it returns all of the users. This is why Tom successfully logged in.

Because there's no input validation (i.e. no checking of legal characters, minimum/maximum string lengths, or removeal of "malicious" characters) Tom has the ability to inject raw SQL syntax within the username and password input fields to alter the meaning of the SQL query responsible for authentication, resulting in a bypass of the application's authentication mechanism - a SQL Inject attack!

So how best to do remediation for the SQL Inject attacks?

Prepared statements (aka parameterized queries) are the best mechanism for preventing SQL injection attacks. Prepared statements are used to abstract SQL statement syntax from input parameters. Statement templates are first defined at the application layer, and the parameters are then passed to them. In Java this can be achieved using a `PreparedStatement` class for sending SQL statements to the backend database. Aside from a better security posture against SQL inject attacks, prepared statements offer improved code quality from a legibility and maintainability perspective due to separation of the SQL logic from its inputs. Here's the modified code:

    String email = request.getParameter("email");
    String password = request.getParameter("password");
    
    // String sql = "select * from users where (email =" + email + " and password =" + password + ")";
    String sql = "select * from users where email = ? and password = ?";
    
    Connection connection = pool.getConnection();
    // Statement statement = connection.createState();
    PreparedStatement preparedStatement = connection.prepareStatement(sql);
    
    // ResultSet result = statement.executeQuery(sql);
    
    preparedStatement.setString(1, email);
    preparedStatement.setString(2, password);
    
    ResultSet result = preparedStatement.executeQuery();
    
    if (result.next()) {
        loggedIn = true;
        # Successfully logged in and redirect to user profile page
    } else {
        # Auth failure - Redirect to Login Page
    }
    
In our modified code example, we first declare the authentication query string and assign it to the `sql` string variable. Notice that the `email` and `password` variable have now been replaced with `?` symbol which act as a place holder for Java's `PreparedStatement` class. The `sql` string variable is then passed as an argument to the `prepareStatement()` method, which precompiles the SQL query and creates a `PreparedStatement` object for sending paramterized SQL statements to the back end SQL server. The `setString()` method is then called to pass the `email` parameter value to our prepared statement. This function takes two arguments, the position index of our placeholder denoted by `?` and the parameter value stored in the the `email` variable. Similarly, the `setString()` method is then called to pass the `password` parameter value to our prepared statement. Finally, we execute our authentication query by invoking the `preparedStatement.executeQuery()` method. The SQL used by `PreparedStatement` is precompiled ensuring that all parameters sent to the underlying database are treated as literal values and not SQL statement/query language, ensuring that no SQL code can be injected using an untrusted parameter.

Ultimately, the security payoff with using prepared statements is that the database will ensure the parameters are automatically escaped.

By default, the JDBCTemplate does its own PreparedStatement internally, if you just use the .update(String sql, Object ... args) form. Spring, and your database, will manage the compiled query for you, so you don't have to worry about opening, closing, resource protection, etc. But still, it is better to know how `PreparedStatement` works.


