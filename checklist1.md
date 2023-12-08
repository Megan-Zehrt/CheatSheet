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



## MY BOILERPLATES

# mysqlconnection
```js
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
# Server File (server.py)
```js

from flask_app import app
from flask_app.controllers import users


#end of the moving content
if __name__ == "__main__":
    app.run(debug=True)

```

# model file (user.py)

```js

#import the function that will return an instance of a connection
from flask_app.config.mysqlconnection import connectToMySQL
# model the class after the user table from our database
from flask_app import DATABASE

class User:
   def __init__(self, data:dict):
      self.id = data['id']
      self.first_name = data['first_name']
      self.last_name = data['last_name']
      self.email = data['email']
      self.created_at = data['created_at']
      self.updated_at = data['updated_at']

      #Add additional columns from database here

   def info(self):
      returnStr = f"First Name = {self.first_name} || Last Name = {self.last_name} || Email = {self.email}"
      return returnStr

#CREATE
   @classmethod
   def create_one(cls, data:dict):
      query = "INSERT INTO users_schema.users (first_name, last_name, email) VALUES (%(first_name)s, %(last_name)s, %(email)s)"
      print("this is the model file")
      result = connectToMySQL(DATABASE).query_db(query, data)
      print(data)
      return result
       
   # now we use the class methods to query our database

#READ
   @classmethod
   def get_all(cls) -> list:
      query = "SELECT * FROM users;"
      #make sure to call the connectToMySQL function with the schema you are targeting

      results = connectToMySQL(DATABASE).query_db(query)
      #create an empty list to append our instances of users
      if not results:
         return []

      instance_list = []
      # iterate over the db results anad create instances of users with cls.
      for dictionary in results:
         instance_list.append(cls(dictionary))
      return instance_list

   @classmethod
   def get_one(cls, data):
      query = "SELECT * FROM users WHERE id = %(id)s;"
      #data = {'id': user_id}
      results = connectToMySQL(DATABASE).query_db(query, data)
      person = cls(results[0])
      return person

   #DELETE
   @classmethod
   def delete(cls, user_id):
      query = "DELETE FROM users WHERE id = %(id)s;"
      data = {"id": user_id}
      return connectToMySQL(DATABASE).query_db(query, data)

   #UPDATE
   @classmethod
   def update(cls, data):
      query = "UPDATE users SET first_name = %(first_name)s, last_name = %(last_name)s, email = %(email)s WHERE id = %(id)s;"
      return connectToMySQL(DATABASE).query_db(query, data)

```

# Controller file (users.py)
```js

from flask_app import app
from flask import render_template,redirect,request,session,flash
from flask_app.models.user import User

# Main Page
@app.route("/users")
def Read():
    all_users = User.get_all()
    return render_template("Read.html", all_users = all_users)

#Create User
@app.route("/users/new")
def new_user():
   return render_template("Create.html")

@app.route("/create_user", methods=['POST'])
def create_user():
   data = {
      'first_name': request.form['first_name'],
      'last_name': request.form['last_name'],
      'email': request.form['email']
   }
   id = User.create_one(data)
   return redirect(f'/users/show/{id}')

# Show User
@app.route("/users/show")
def user_show():
   return render_template("Show.html")

@app.route("/users/show/<int:id>")
def user_show_id(id):
   data = {
      'id': id
   }
   one_user= User.get_one(data)
   print(one_user)
   return render_template("Show.html", one_user=one_user)

# Delete User
@app.route("/users/delete/<int:user_id>")
def delete_user(user_id):
   User.delete(user_id)
   return redirect("/users")

# Edit User
@app.route("/users/edit/<int:id>")
def edit_user(id):
   data = {
      'id': id
   }
   one_user = User.get_one(data)
   return render_template("Edit.html", one_user=one_user)


# Update User

@app.route("/users/update/<int:id>", methods=['POST'])
def update_user(id):
   data = {
      'id': id,
      **request.form
   }
   User.update(data)
   return redirect("/users")

```

# Dunder Init File (__init__.py)

```js

from flask import Flask, render_template, redirect, request
app = Flask(__name__)
app.secret_key = 'secret'
DATABASE = "users_schema"

```

## VALIDATION BOILERPLATES

# Server file (server.py)

```js

from flask_app import app
from flask_app.controllers import controller_login
app.secret_key = 'the secret key, key the secret of the secret key I think?'


#end of the moving content
if __name__ == "__main__":
    app.run(debug=True)

```

# Model File (model_user.py)
```js

#import the function that will return an instance of a connection
from flask_app.config.mysqlconnection import connectToMySQL
from flask import flash
# model the class after the user table from our database
from flask_app import DATABASE
import re

EMAIL_REGEX = re.compile(r'^[a-zA-Z0-9.+_-]+@[a-zA-Z0-9._-]+\.[a-zA-Z]+$') 

class User:
   def __init__(self, data:dict):
      self.id = data['id']
      self.first_name = data['first_name']
      self.last_name = data['last_name']
      self.email = data['email']
      self.password = data['password']
      self.created_at = data['created_at']
      self.updated_at = data['updated_at']

      #Add additional columns from database here

   def info(self):
      returnStr = f"First Name = {self.first_name} || Last Name = {self.last_name} || Email = {self.email} Password = {self.password}"
      return returnStr

#CREATE
   @classmethod
   def create_one(cls, data:dict):
      query = "INSERT INTO users (first_name, last_name, email, password) VALUES (%(first_name)s, %(last_name)s, %(email)s, %(password)s)"
      print("this is the model file")
      result = connectToMySQL(DATABASE).query_db(query, data)
      print(data)
      return result
       
   # now we use the class methods to query our database 

#READ
   @classmethod
   def get_all(cls) -> list:
      query = "SELECT * FROM users;"
      #make sure to call the connectToMySQL function with the schema you are targeting

      results = connectToMySQL(DATABASE).query_db(query)
      #create an empty list to append our instances of users
      if not results:
         return []

      instance_list = []
      # iterate over the db results anad create instances of users with cls.
      for dictionary in results:
         instance_list.append(cls(dictionary))
      return instance_list

   @classmethod
   def get_one(cls, data):
      query = "SELECT * FROM users WHERE id = %(id)s;"
      #data = {'id': user_id}
      results = connectToMySQL(DATABASE).query_db(query, data)
      if not results:
         return []

      instance_list = []

      for dictionary in results:
         instance_list.append(cls(dictionary))
      return instance_list


   @classmethod
   def get_by_email(cls,data):
      query = "SELECT * FROM users WHERE email = %(email)s;"
      result = connectToMySQL(DATABASE).query_db(query,data)
      # Didn't find a matching user
      if len(result) < 1:
         return False
      return cls(result[0])


# Validators

   @staticmethod
   def validator(data: dict) -> bool:
      is_valid = True

      if(len(data['first_name']) < 2):
         flash("First Name must be more than 2 Characters in length", "err_users_first_name")
         is_valid = False

      if(len(data['last_name']) < 2):
         flash("Last Name must be more than 2 Characters in length", "err_users_last_name")
         is_valid = False

      if(len(data['email'])) == 0:
         flash("Not a good Email", "err_users_email")
         is_valid = False

      elif not EMAIL_REGEX.match(data['email']):
         flash("Invalid email address", "err_users_email")
         is_valid = False


      if(len(data['password']) < 8):
         flash("Not a good password", "err_users_password")
         is_valid = False

      if(len (data['confirm_password']) < 8):
         flash("Confirm Password is not Valid", "err_users_confirm_password")
         is_valid = False

      elif((data['confirm_password']) != (data['password'])):
         flash("Confirm Password must Match Password", "err_users_confirm_password")
         is_valid = False


      return is_valid
      #run through some if checks -> if if checks come out to be bad then is_valid = False

# Validators

   @staticmethod
   def validator_login(data: dict) -> bool:
      is_valid = True

      if(len(data['email'])) == 0:
         flash("Invalid Email", "err_users_loginemail")
         is_valid = False

      elif not EMAIL_REGEX.match(data['email']):
         flash("Invalid email address", "err_users_loginemail")
         is_valid = False


      if(len(data['password']) < 8):
         flash("Invalid password", "err_users_loginpassword")
         is_valid = False




      return is_valid

# SAVE
   @classmethod
   def save(cls,data):
      query = "INSERT INTO users (first_name, last_name, email, password) VALUES (%(first_name)s, %(last_name)s, %(email)s, %(password)s);"
      result = connectToMySQL(DATABASE).query_db(query, data)
      return result

```

# Controller File (controller_user.py)
```js

from flask_app import app
from flask import render_template,redirect,request,session,flash, url_for
from flask_app.models.model_user import User
from flask_bcrypt import Bcrypt        
bcrypt = Bcrypt(app)     # we are creating an object called bcrypt, 
                         # which is made by invoking the function Bcrypt with our app as an argument



# Login Session
@app.route("/login", methods=['POST'])
def login_process():
   if request.method == "POST":
      session['email'] = request.form['email']
      session['password'] = request.form['password']
      if not User.validator_login(request.form):
        return redirect ('/')

      # see if the username provided exists in the database
      data = { "email" : request.form["email"] }
      user_in_db = User.get_by_email(data)
      print(user_in_db.password)
      print(session['password'])
      # user is not registered in the db
      if not user_in_db:
          flash("Invalid Email", "err_users_loginemail")
          return redirect("/")
      if not bcrypt.check_password_hash(user_in_db.password, request.form['password']):
          flash("Invalid Password", "err_users_loginpassword")
          return redirect('/')
      # if the passwords matched, we set the user_id into session
      session['user_id'] = user_in_db.id
      return redirect('/dashboard')

  # Main Page
@app.route("/")
def home():
  if "user_id" in session:
    return redirect('/dashboard')
  return render_template("Home.html")

# Register Session
@app.route('/register', methods=['POST'])
def submit_form():

        is_valid = User.validator(request.form)

        if not is_valid:
            return redirect('/')

        pw_hash = bcrypt.generate_password_hash(request.form['password'])
        print(pw_hash)
        # put the pw_hash into the data dictionary
        data = {
            "first_name": request.form['first_name'],
            "last_name": request.form['last_name'],
            "email": request.form['email'],
            "password" : pw_hash
        }

        if request.method == 'POST':
          session['first_name'] = request.form['first_name']
          session['last_name'] = request.form.get('last_name')
          session['email'] = request.form.get('email')
          # Call the save @classmethod on User
        user_id = User.save(data)
        # store user id into session
        session['user_id'] = user_id
        return redirect('/dashboard')


@app.route("/dashboard")
def dashbaord():
  if not "user_id" in session:
    return redirect('/')
  print(session['user_id'])
  data = {
    'id': session['user_id']
  }
  user = User.get_one(data)
  return render_template("Welcome.html", user = user)

@app.route('/logout')
def logout():
  session.pop('user_id')
  return redirect("/")

```

# HTML Form File (Form.html)

```h

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" type="text/css" href="{{url_for('.static', filename='css/style.css')}}">
    <title>Document</title>
</head>
<body>
    <div class="mainpage">
        <div class="registerbox">
            <p class="headerR">Register</p>

             
            <form action="/register", method="post">

                <div class="label">
                    <label for="first_name">First Name</label>
                    <input type="text" name="first_name" id="first_name">
                        {% for message in get_flashed_messages(category_filter=['err_users_first_name']) %}
                        <p class="errmsg">{{message}}</p>
                        {% endfor %}
                </div>
                <div class="label">
                    <label for="last_name">Last Name</label>
                    <input type="text" name="last_name" id="last_name">
                    {% for message in get_flashed_messages(category_filter=['err_users_last_name']) %}
                    <p class="errmsg">{{message}}</p>
                    {% endfor %}
                </div>
                <div class="label">
                    <label for="email">Email</label>
                    <input type="text" name="email" id="email">
                    {% for message in get_flashed_messages(category_filter=['err_users_email']) %}
                    <p class="errmsg">{{message}}</p>
                    {% endfor %}
                </div>
                <div class="label">
                    <label for="password">Password</label>
                    <input type="text" name="password" id="password">
                    {% for message in get_flashed_messages(category_filter=['err_users_password']) %}
                    <p class="errmsg">{{message}}</p>
                    {% endfor %}
                </div>
                <div class="label">
                    <label for="confirm_password">Confirm Password</label>
                    <input type="text" name="confirm_password" id="confirm_password">
                    {% for message in get_flashed_messages(category_filter=['err_users_confirm_password']) %}
                    <p class="errmsg">{{message}}</p>
                    {% endfor %}
                </div>
                <button class="buttonR">Register</button>
    
            </form>
            
        </div>

        <div class="loginbox">
            <p class="headerL">Login</p>

            <form action="/login", method="post">
                <div class="label">
                    <label for="email">Email</label>
                    <input type="text" name="email" id="email">
                    {% for message in get_flashed_messages(category_filter=['err_users_loginemail']) %}
                    <p class="errmsg">{{message}}</p>
                    {% endfor %}
                </div>
                <div class="label">
                    <label for="password">Password</label>
                    <input type="text" name="password" id="password">
                    {% for message in get_flashed_messages(category_filter=['err_users_loginpassword']) %}
                    <p class="errmsg">{{message}}</p>
                    {% endfor %}
                </div>
                <button class="buttonL">Login</button>
            </form>
        </div>
    </div>
    <script src="{{url_for('static', filename='script.js')}}"></script>
</body>
</html>

```

