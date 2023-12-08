## BOILERPLATES

# SERVER.PY

```js

from flask_app import app
from flask_app.controllers import controller_user, controller_recipe
app.secret_key = 'the secret key, key the secret of the secret key I think?'


#end of the moving content
if __name__ == "__main__":
    app.run(debug=True)

```

# __init__.py

```js

from flask import Flask, render_template, redirect, request, session, flash
app = Flask(__name__)
app.secret_key = 'secret'
DATABASE = "recipe_schema"

```

# controller_recipe.py

```js

from flask_app import app
from flask import render_template,redirect,request,session,flash, url_for
from flask_app.models.model_recipe import Recipe
from flask_app.models.model_user import User
from flask_bcrypt import Bcrypt        
bcrypt = Bcrypt(app)     # we are creating an object called bcrypt, 
                         # which is made by invoking the function Bcrypt with our app as an argument




@app.route("/recipe/new")
def new_recipe():
    if not "user_id" in session:
        return redirect('/')

        recipes = Recipe.get_all()
    recipes= Recipe.get_all()
    return render_template("new.html", recipes=recipes)

@app.route("/create_recipe", methods=['POST'])
def create_recipe():
    is_valid = Recipe.validator(request.form)
    if not is_valid:
        return redirect('/recipe/new')
    data = {
      'name': request.form['name'],
      'description': request.form['description'],
      'instructions': request.form['instructions'],
      'time': request.form['time'],
      'under': request.form['under'],
      'user_id': session['user_id']
    }
    id = Recipe.create_one(data)
    return redirect("/dashboard")

# Show Recipe

@app.route("/recipe/<int:id>")
def recipe_show_id(id):
    print("*********",id,"**********")
    if not "user_id" in session:
      return redirect('/')

    recip = Recipe.get_recipes()
    recipes = Recipe.get_one_recipe({ 'id': id})
    user = User.get_one({ 'id': session['user_id']})
    return render_template("Show_recipe.html", recipes = recipes, user = user, recip=recip )

# Delete Recipe
@app.route("/recipe/delete/<int:recipe_id>")
def delete_recipe(recipe_id):
   data = {
      'id': recipe_id
   }
   Recipe.delete(data)
   return redirect("/dashboard")

# Edit Recipe

@app.route("/recipe/edit/<int:id>")
def edit_recipe(id):
  if not "user_id" in session:
    return redirect('/')

    is_valid = Recipe.validator(request.form)
    if not is_valid:
        return redirect('/recipe/edit')

  data = {"id" : id}
  recipes = Recipe.get_one_recipe(data)
  print(recipes)
  user = User.get_one({ 'id': session['user_id']})
  return render_template("Recipe_edit.html", recipes = recipes, user=user)


@app.route("/recipe/edit", methods=['POST'])
def update_recipe():
    print("\n\n\n\n\n******************************", request.form)
    is_valid = Recipe.validator(request.form)
    if not is_valid:
        return redirect(f"/recipe/edit/{request.form['id']}")
    data = {
      'id' : request.form['id'],
      'name': request.form['name'],
      'description': request.form['description'],
      'instructions': request.form['instructions'],
      'time': request.form['time'],
      'under': request.form['under'],
      'user_id': session['user_id']
    }
    Recipe.update(data)
    return redirect("/dashboard")

```

# controller_user.py

```js

from flask_app import app
from flask import render_template,redirect,request,session,flash, url_for
from flask_app.models.model_user import User
from flask_app.models.model_recipe import Recipe
from flask_bcrypt import Bcrypt        
bcrypt = Bcrypt(app)     # we are creating an object called bcrypt, 
                         # which is made by invoking the function Bcrypt with our app as an argument




@app.route("/dashboard")
def dashbaord():
  if not "user_id" in session:
    return redirect('/')
  print(session['user_id'])

  recipes = Recipe.get_recipes()
  user = User.get_one({ 'id': session['user_id']})
  return render_template("Welcome.html", recipes = recipes, user=user)

@app.route('/logout')
def logout():
  session.pop('user_id')
  return redirect("/")


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

```

# model_recipe.py

```js

#import the function that will return an instance of a connection
from flask_app.config.mysqlconnection import connectToMySQL
from flask import flash
from flask_app.models import model_user
# model the class after the user table from our database
from flask_app import DATABASE
import re

EMAIL_REGEX = re.compile(r'^[a-zA-Z0-9.+_-]+@[a-zA-Z0-9._-]+\.[a-zA-Z]+$') 

class Recipe:
   def __init__(self, data:dict):
      self.id = data['id']
      self.name = data['name']
      self.description = data['description']
      self.instructions = data['instructions']
      self.time = data['time']
      self.under = data['under']
      self.user_id = data['user_id']
      self.created_at = data['created_at']
      self.updated_at = data['updated_at']

      #Add additional columns from database here

   def info(self):
      returnStr = f"First Name = {self.name} || Last Name = {self.description} || instructions = {self.instructions} time = {self.time}, under = {self.under}"
      return returnStr

#CREATE
   @classmethod
   def create_one(cls, data:dict):
      query = "INSERT INTO recipes (name, description, instructions, time, under, user_id) VALUES (%(name)s, %(description)s, %(instructions)s, %(time)s, %(under)s, %(user_id)s)"
      print("this is the model file")
      result = connectToMySQL(DATABASE).query_db(query, data)
      print(data)
      return result
       
   # now we use the class methods to query our database 

#READ
   @classmethod
   def get_all(cls) -> list:
      query = "SELECT * FROM recipes;"
      #make sure to call the connectToMySQL function with the schema you are targeting

      results = connectToMySQL(DATABASE).query_db(query)
      #create an empty list to append our instances of recipes
      if not results:
         return []

      instance_list = []
      # iterate over the db results anad create instances of recipes with cls.
      for dictionary in results:
         instance_list.append(cls(dictionary))
      return instance_list

   @classmethod
   def get_one(cls, data):
      query = "SELECT * FROM recipes WHERE recipes.id = %(id)s;"
      print(query)
      results = connectToMySQL(DATABASE).query_db(query, data)
      if not results:
         return []

      instance_list = []

      for dictionary in results:
         instance_list.append(cls(dictionary))
      return instance_list


   @classmethod
   def get_by_email(cls,data):
      query = "SELECT * FROM recipes WHERE email = %(email)s;"
      result = connectToMySQL(DATABASE).query_db(query,data)
      # Didn't find a matching user
      if len(result) < 1:
         return False
      return cls(result[0])


# Validators

   @staticmethod
   def validator(data: dict) -> bool:
      is_valid = True

      if(len(data['name']) < 2):
         flash("First Name must be more than 2 Characters in length", "err_recipes_name")
         is_valid = False

      if(len(data['description']) < 1):
         flash("Description must be more than 1 Characters in length", "err_recipes_description")
         is_valid = False

      if(len(data['instructions']) < 1):
         flash("Instructions must be more than 1 character in length", "err_recipes_instructions")
         is_valid = False


      if(len(data['time'])) ==0:
         flash("A date needs to be selected", "err_recipes_time")
         is_valid = False

      if 'under' not in data:
         flash("Did the recipe take longer than 30 minutes to make?", "err_recipes_under")
         is_valid = False


      return is_valid
      #run through some if checks -> if if checks come out to be bad then is_valid = False



# SAVE
   @classmethod
   def save(cls,data):
      query = "INSERT INTO recipes (name, description, email, password) VALUES (%(name)s, %(description)s, %(email)s, %(password)s);"
      result = connectToMySQL(DATABASE).query_db(query, data)
      return result

   #DELETE
   @classmethod
   def delete(cls, data):
      query = "DELETE FROM recipes WHERE id = %(id)s;"
      return connectToMySQL(DATABASE).query_db(query, data)

   #UPDATE
   @classmethod
   def update(cls, data):
      query = "UPDATE recipes SET name = %(name)s, description = %(description)s, instructions = %(instructions)s, time = %(time)s, under = %(under)s WHERE id = %(id)s;"
      return connectToMySQL(DATABASE).query_db(query, data)

# for the show route to show the one recipe that you selected on the "view recipe" URL
   @classmethod
   def get_one_recipe(cls, data):
      query = "SELECT * FROM recipes JOIN users ON users.id = recipes.user_id WHERE recipes.id = %(id)s;"
      print(query)
      results = connectToMySQL(DATABASE).query_db(query, data)
      if results:

         one_recipe = cls(results[0])
         for dictionary in results:

            users_data = {
               **dictionary,
               "id": dictionary["users.id"],
               "updated_at": dictionary["users.updated_at"],
               "created_at": dictionary["users.created_at"]
            }
            print(users_data)

            u = model_user.User(users_data)
            one_recipe.u = u
         return one_recipe

# for the display route where it shows the user that created the recipe in the "posted by" coloumn
   @classmethod
   def get_recipes(cls):
      query = "SELECT * FROM recipes JOIN users ON users.id = recipes.user_id;"
      print(query)
      results = connectToMySQL(DATABASE).query_db(query)
      if results:

         instance_list = []

         for dictionary in results:
            one_recipe = cls(dictionary)

            users_data = {
               **dictionary,
               "id": dictionary["users.id"],
               "updated_at": dictionary["users.updated_at"],
               "created_at": dictionary["users.created_at"]
            }
            print(users_data)

            u = model_user.User(users_data)
            one_recipe.u = u
            instance_list.append(one_recipe)
         return instance_list

```

# model_user.py

```js
#import the function that will return an instance of a connection
from flask_app.config.mysqlconnection import connectToMySQL
from flask import flash
from flask_app.models import model_recipe
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
      
      else:
         potential_user = User.get_by_email(data)
         if potential_user:
            flash("This Email already exists", "err_users_email")
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

