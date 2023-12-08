1. mkdir hello_flask
2. cd hello_flask
3. pip install pipenv      #only for new computor. (install only once)
4. pipenv install flask pymysql (whats in your env run: pip list)
5. pipenv shell

6. create my file structure
   1. Root folder (name of my projects)
     1. flask_app
        1. config 📂
           1. mysqlconnection.py 📄
        2. controllers 📂
           1. controller_animal.py 📄
        3. models 📂
           1. model_animal.py 📄
        4. static 📂
           1. css 📂
              1. style.css 📄
           2. js 📂
              1. script.js 📄
        5. templates 📂
           1. index.html 📄
        6. \_\_init__.py 📄
     2. pipfile 📄
     3. pipfile.lock 📄
     4. server.py 📄
7. add boilerplate code 
8. Test to make sure server is working













# My File boilerplates
# dunder init file
```py
from flask import Flask #Import Flask to allow us to create our app
app = Flask(__name__) # create a new instance of the Flask class called "app"
app.secret_key = 'the secret key'
DATABASE = "your DATABASE name here"

```
## server.py
```py
from flask_app import app
#for every controller file, add below "controller_name, controller_name2, ..."
from flask_app.controllers import controller_name
# this is always at the bottom!
if __name__=="__main__":    # ensure this file being run directly and not from a different module
   app.run(debug=True)    # run the app in debug mode.

#OR

from flask import Flask, render_template, redirect, request
app = Flask(__name__)
app.secret_key = 'secret'

@app.route('/')
def index():
    return render_template("index.html")

if __name__ == "__main__":
    app.run(debug=True)


```
## controller.py
```py
from flask import render_template, redirect, request, session
from flask_app import app
from flask_app.models.model_name import Classname

# the route structure that we are after is /table name/id? if its (avaiblible)/ action

@app.route('/')
def hello_world():
   return 'Hello World'

```

## mysqlconnection.py
```py
# a cursor is the object we use to interact with the database
import pymysql.cursors
# this class will give us an instance of a connection to our database
class MySQLConnection:
    def __init__(self, db):
        # change the user and password as needed
        connection = pymysql.connect(host = 'localhost',
                                    user = 'root', 
                                    password = 'root', 
                                    db = db,
                                    charset = 'utf8mb4',
                                    cursorclass = pymysql.cursors.DictCursor,
                                    autocommit = False)
        # establish the connection to the database
        self.connection = connection
    # the method to query the database
    def query_db(self, query:str, data:dict=None):
        with self.connection.cursor() as cursor:
            try:
                query = cursor.mogrify(query, data)
                print("Running Query:", query)
     
                cursor.execute(query)
                if query.lower().find("insert") >= 0:
                    # INSERT queries will return the ID NUMBER of the row inserted
                    self.connection.commit()
                    return cursor.lastrowid
                elif query.lower().find("select") >= 0:
                    # SELECT queries will return the data from the database as a LIST OF DICTIONARIES
                    result = cursor.fetchall()
                    return result
                else:
                    # UPDATE and DELETE queries will return nothing
                    self.connection.commit()
            except Exception as e:
                # if the query fails the method will return FALSE
                print("Something went wrong", e)
                return False
            finally:
                # close the connection
                self.connection.close() 
# connectToMySQL receives the database we're using and uses it to create an instance of MySQLConnection
def connectToMySQL(db):
    return MySQLConnection(db)
```
## model.py file
```py
#import the function that will return an instance of a connection
from flask_app.config.mysqlconnection import connectToMySQL
from flask app import DATABASE
# model the class after the friend table from our database
class Friend:
   def __init__(self, data:dict):
      self.id = data['id']
      self.created_at = data['created_at']
      self.updated_at = data['updated_at']

      #Add additional columns from database here
      
   # now we use the class methods to query our database
   @classmethod
   def get_all(cls):
      query = "SELECT * FROM friends;"
      #make sure to call the connectToMySQL function with the schema you are targeting

      results = connectToMySQL(DATABASE).query_db(query)
      #create an empty list to append our instances of friends
      all_friends = []
      # iterate over the db results anad create instances of friends with cls.
      for friend in results:
         all_friends.append(cls(friend))
      return all_friends

```

```
connecting a css page in your html file
<link rel="stylesheet" type="text/css" href="{{url_for('static', filename='css/style.css')}}">

```

```
CONNECTING TO A SCRIPT TAG
<script src="{{url_for('static', filename='js/script.js')}}">
```