---
layout: post
title:  "Connecting Python and MySQL"
tag: python mysql databases
---


# Setting Up Databases with python
First one has to install `MySQL-Server`. A easy option is the `MySQL Workbench` which is a nice tool to manage and monitore the databases. During the installation of `Workbench` everything that one need will be installed.

> Note: NEVER LOOSE YOUR ROOT PASSWORD. This is a grown up tool. If you loose your root credentials you might not be able to connect to your database again.

## Setting up a database
Now lets start with creating a new database. After starting Workbench you will see all your available connections. There should at least be the `local` one (so the one that runs on YOUR machine), but you could also connect to other databases on different servers.
Open your local connection

![Overview Workbench](jekyll-blog/assets/SQL_Worbench_Overview.JPG)
On the left you can see you databases. To create a new one you can left-click in that area. With left-click on `tables` you can create a new table. The right side is the `Console` here you can write and execute SQL commands. Feel free to use it for creations of databases as well.

To Query things from the DB we first tell SQL which database to use with `USE testdb`. `Show databases` gives us a possible choices.

In our case we want to create a table for our tweets. We create an ID column and select `AI` for autoincrement and `NN` so is required to have it. For the tweet we have some design options. We can either store the string in a `LARGETEXT` columns or the `json` object in a `BLOB`. Note that since wuite a while MySQL supports `JSON` as a own datatype so maybe we should use that one.

![Creating a Table](/assets/New_table.JPG)

### Creating a User
Since we might not use the root user inside our data application we should create a new user for the database.
![Adding Users](/assets/adding_users.JPG)
Under `Server > Users and Privileges` we can select `Add Account` and then use a name and choose privileges. It is always a good idea to use the minimal required privileges just in case someone hacks that user.


### Python and MySql
We are here inheriting the `TweetListener` class from `tweepy` and overwrite the `on_data` method, but that is not too much important. Every time a new tweet comes in the method gets called and we want to implement it in such way that the tweets are stored in the database. To connect to our database we need a package that handles that. A common choice seems to be `mysql.connector`. After importing it we can use the `create_connection` method to authenticate and connect to the DB.
```python
def create_connection(host_name, user_name, user_password, db_name):
    connection = None
    connection = mysql.connector.connect(
        host=host_name,
        user=user_name,
        passwd=user_password,
        database=db_name,auth_plugin='mysql_native_password',
        use_unicode = True,

    )
    #this one is important to prevent incorrect string value
    #https://dev.mysql.com/doc/connector-python/en/connector-python-api-mysqlconnection-set-charset-collation.html
    connection.set_charset_collation(charset="utf8mb4", collation=None)
    return connection
```

Unfortunately mysql.connector does not support conetxt manager so we can not use the `with` command, but have to use `try:` and `finally:` to ensure that connections are closed properly in case of an error. (Note: statements in `finally:` are executed under with or without any exceptions)

Since we are taking Input more or less directly from the user, we have to make sure that we sanitize the input. Apparently by using the `?` notation and a tuple, `sql.connetor` takes care of that (Note sure though). Other connection modules might offer explicit sanitation methods such as: ` escaped_string = db.escape_string('test string')`. But from my current understanding this is the same method which is automatically called by using tuple and `?`
```python
def on_data(self, tweet):
        try:
            text= json.loads(tweet).get("text",'EMPTY')
            connection = create_connection("localhost", username, password, tablename)
            if connection.is_connected():
                cursor = connection.cursor()
                cursor.execute('''INSERT INTO tweets (json) VALUES ("?")''',(text,))
                cursor.execute('''INSERT INTO tweets (json) VALUES ("%s")''', (text,))
                connection.commit()
                self.c +=1
        except Error as e :
            print ("Error while connecting to MySQL", e)
        finally:
            cursor.close()
            connection.close()
```

Note that data is only safely written after we commit the connection.

### Changing the charset
Apparently the unicode that is used in twitter posts is different from the default SQL one. To fix this we want to edit the table once again. We select the column that stores the text and change the charset to `utf8mb4` and the collation to `utf8mb4_unicode_ci`.

In the connector funcrio we also have to specify the encoding with: 
```python
    #this one is important to prevent incorrect string value
    #https://dev.mysql.com/doc/connector-python/en/connector-python-api-mysqlconnection-set-charset-collation.html
    connection.set_charset_collation(charset="utf8mb4", collation=None)
```