brew install postgresql

brew tap homebrew/services

brew services start postgresql


psql
create user musab;
alter user musab with encrypted password '123qwe';
create database sampleDB1;
grant all privileges on database sampledb1 to musab;
\q
Python Flask example:



from flask import Flask, request
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = "postgresql://musab:123qwe@localhost:5432/sampledb1"
db = SQLAlchemy(app)
db.create_all()
migrate = Migrate(app, db)


class CarsModel(db.Model):
   __tablename__ = 'cars'

   id = db.Column(db.Integer, primary_key=True)
   name = db.Column(db.String())
   model = db.Column(db.String())
   doors = db.Column(db.Integer())

   def __init__(self, name, model, doors):
       self.name = name
       self.model = model
       self.doors = doors

   def __repr__(self):
       return f"<Car {self.name}>"


@app.route('/cars', methods=['POST', 'GET'])
def handle_cars():
   if request.method == 'POST':
       if request.is_json:
           data = request.json
           new_car = CarsModel(name=data['name'], model=data['model'], doors=data['doors'])
           print("new_car")
           db.session.add(new_car)
           db.session.commit()
           return {"message": f"car {new_car.name} has been created successfully."}
       else:
           return {"error": "The request payload is not in JSON format"}

   elif request.method == 'GET':
       cars = CarsModel.query.all()
       results = [
           {
               "id": car.id,
               "name": car.name,
               "model": car.model,
               "doors": car.doors
           } for car in cars]

       return {"count": len(results), "cars": results}


@app.route('/cars/<car_id>', methods=['GET', 'PUT', 'DELETE'])
def handle_car(car_id):
   car = CarsModel.query.get_or_404(car_id)

   if request.method == 'GET':
       response = {
           "name": car.name,
           "model": car.model,
           "doors": car.doors
       }
       return {"message": "success", "car": response}

   elif request.method == 'PUT':
       data = request.get_json()
       car.name = data['name']
       car.model = data['model']
       car.doors = data['doors']
       db.session.add(car)
       db.session.commit()
       return {"message": f"car {car.name} successfully updated"}

   elif request.method == 'DELETE':
       db.session.delete(car)
       db.session.commit()
       return {"message": f"Car {car.name} successfully deleted."}





With Authentication:


from os import abort

from flask import Flask, request, jsonify, url_for, json, g
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from passlib.apps import custom_app_context as pwd_context
from flask_httpauth import HTTPBasicAuth

auth = HTTPBasicAuth()
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = "postgresql://musab:123qwe@localhost:5432/sampledb1"
db = SQLAlchemy(app)
db.create_all()
migrate = Migrate(app, db)


class CarsModel(db.Model):
   __tablename__ = 'cars'

   id = db.Column(db.Integer, primary_key=True)
   name = db.Column(db.String())
   model = db.Column(db.String())
   doors = db.Column(db.Integer())

   def __init__(self, name, model, doors):
       self.name = name
       self.model = model
       self.doors = doors

   def __repr__(self):
       return f"<Car {self.name}>"


class User(db.Model):
   __tablename__ = 'users'
   id = db.Column(db.Integer, primary_key=True)
   username = db.Column(db.String(32), index=True)
   password_hash = db.Column(db.String(128))

   def hash_password(self, password):
       self.password_hash = pwd_context.encrypt(password)

   def verify_password(self, password):
       return pwd_context.verify(password, self.password_hash)


@app.route('/cars', methods=['POST', 'GET'])
@auth.login_required()
def handle_cars():
   if request.method == 'POST':
       if request.is_json:
           data = request.json
           new_car = CarsModel(name=data['name'], model=data['model'], doors=data['doors'])
           print("new_car")
           db.session.add(new_car)
           db.session.commit()
           return {"message": f"car {new_car.name} has been created successfully."}
       else:
           return {"error": "The request payload is not in JSON format"}

   elif request.method == 'GET':
       cars = CarsModel.query.all()
       results = [
           {
               "id": car.id,
               "name": car.name,
               "model": car.model,
               "doors": car.doors
           } for car in cars]

       return {"count": len(results), "cars": results}


@app.route('/cars/<car_id>', methods=['GET', 'PUT', 'DELETE'])
@auth.login_required
def handle_car(car_id):
   car = CarsModel.query.get_or_404(car_id)

   if request.method == 'GET':
       response = {
           "name": car.name,
           "model": car.model,
           "doors": car.doors
       }
       return {"message": "success", "car": response}

   elif request.method == 'PUT':
       data = request.get_json()
       car.name = data['name']
       car.model = data['model']
       car.doors = data['doors']
       db.session.add(car)
       db.session.commit()
       return {"message": f"car {car.name} successfully updated"}

   elif request.method == 'DELETE':
       db.session.delete(car)
       db.session.commit()
       return {"message": f"Car {car.name} successfully deleted."}


@app.route('/api/users', methods=['POST', 'GET'])
def new_user():
   if request.method == 'POST':
       username = request.json.get('username')
       password = request.json.get('password')
       if username is None or password is None:
           abort(400)  # missing arguments
       if User.query.filter_by(username=username).first() is not None:
           abort(400)  # existing user
       user = User(username=username)
       user.hash_password(password)
       db.session.add(user)
       db.session.commit()
       return jsonify({'username': user.username}), 201, {'Location': url_for('get_user', id=user.id, _external=True)}

   elif request.method == 'GET':
       users = User.query.all()
       results = [
           {
               "id": user.id,
               "username": user.username,
               "password": user.password_hash
           } for user in users]

       return jsonify({"users": results})


@auth.verify_password
def verify_password(username, password):
   user = User.query.filter_by(username=username).first()
   if not user or not user.verify_password(password):
       return False
   g.user = user
   return True

