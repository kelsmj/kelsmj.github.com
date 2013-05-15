---
layout: post
title: "Test Driven Development of a Flask API"
date: 2013-05-15 09:21
comments: true
categories: python flask test
---

Recently, I have been fiddling around with [Flask](http://flask.pocoo.org/) to create some Restful API's.  I found when developing these API's, instead of using and abusing CURL commands to test the API, it was easiest to write Unit Tests as I went along in order to test and verify the routes I had were working.  The following is a commentary on how I set it all up and got it running.

The packages I used for this setup are:

* [Flask](http://flask.pocoo.org/) (pip install flask)
* [Flask-Restful](http://flask-restful.readthedocs.org/) (pip install flask-restful)
* [Nose](https://nose.readthedocs.org/en/latest/) (pip install nose)
* [SQLAlchemy](http://www.sqlalchemy.org/) (pip install sqlalchemy)
* [Flask-SQLAlchemy](http://pythonhosted.org/Flask-SQLAlchemy/) (pip install flask-sqlalchemy)
* [Psycopg2](http://initd.org/psycopg/) (pip install psycopg2)

First, lets create a simple postgresql database.  Since I want the test database to be separate from the main database, I am going to create one called **flaskexample** and one called **flaskexample_test**.  I will show you later on, how to specify the test database when running the nose tests.

{% codeblock console %}
$ createuser flaskexample -P -d
  Enter password for new role:
  Enter it again:
$ createdb flaskexample -U flaskexample -h localhost
$ createdb flaskexample_test -U flaskexample -h localhost
{% endcodeblock %}

Now, we can create our simple flask application.  For the purposes of this blog, the flask application will only have a User table that we will go against.  Copy the code below into your application.py file, then in the console start the python interpreter and run the following:

{% codeblock console %}
>>> from application import init_db
>>> init_db()
{% endcodeblock %}

If this runs successfully, this should create your users table in your flaskexample database.

Now for the main Flask application.  In this contrived example, we will have 2 routes to get to our user data.  We will have the */users* route, where we can either **get** a list of users or **post** a new user.  We also have the */users/\<string:id>* route where we will be able to get a single user.  Our tests will center around getting a list of users, adding a user, getting a specific user, trying to add a user when an email already exists and finally deleting that user.

{% codeblock lang:python %}
import os
from flask import Flask
from flask.ext import restful
from flask.ext.restful import Resource,  reqparse, Api
from sqlalchemy import create_engine, Column, String, Integer
from sqlalchemy.orm import scoped_session, sessionmaker, relationship, column_property
from sqlalchemy.ext.declarative import declarative_base, declared_attr
from sqlalchemy.exc import IntegrityError

app = Flask("flasktestexample")
api = Api(app)
app.debug = True

if os.environ.get('DATABASE_URL') is None:
  engine = create_engine('postgres://flaskexample:flask@localhost:5432/flaskexample', convert_unicode=True)
else:
  engine = create_engine(os.environ['DATABASE_URL'], convert_unicode=True)

db_session = scoped_session(sessionmaker(autocommit=False,
                                         autoflush=False,
                                         bind=engine))
Base = declarative_base()
Base.query = db_session.query_property()

@app.teardown_request
def teardown_request(exception):
    db_session.remove()


def init_db():
    Base.metadata.drop_all(bind=engine)
    Base.metadata.create_all(bind=engine)

#User Model
class User(Base):
    __tablename__ = 'users'

    #from http://stackoverflow.com/a/11884806
    def as_dict(self):
      return {c.name: getattr(self, c.name) for c in self.__table__.columns}

    id = Column(Integer, primary_key=True)
    first_name = Column(String(200))
    last_name = Column(String(200))
    email = Column(String(200), unique=True)

#Parser arguments that Flask-Restful will check for
parser = reqparse.RequestParser()
parser.add_argument('first_name', type=str, required=True, help="First Name Cannot Be Blank")
parser.add_argument('last_name', type=str, required=True, help="Last Name Cannot Be Blank")
parser.add_argument('email', type=str, required=True, help="Email Cannot Be Blank")

#Flask Restful Views
class UserView(Resource):
  def get(self, id):
    e = User.query.filter(User.id == id).first()
    if e is not None:
      return e.as_dict()
    else:
      return {}


class UserViewList(Resource):
  def get(self):
    e = User.query.all()
    results = []
    for row in User.query.all():
      results.append(row.as_dict())
    return results

  def post(self):
      args = parser.parse_args()
      o = User()
      o.first_name = args["first_name"]
      o.last_name = args["last_name"]
      o.email = args["email"]

      try:
        db_session.add(o)
        db_session.commit()
      except IntegrityError, exc:
        return {"error": exc.message}, 500

      return o.as_dict(), 201

#Flask Restful Routes
api.add_resource(UserViewList, '/users')
api.add_resource(UserView, '/users/<string:id>')

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)


{% endcodeblock %}

For the nose tests, I create a **tests** directory, under my application root.  For this example, we will have two files.  A **__init__.py** file, where we will setup the flask test client and the SQLAlchemy database session.  I also have a **testusers.py** file, which is where all the tests will go.  See the code blocks below.

**__init__.py**

{% codeblock lang:python %}
import os
import base64
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker
from sqlalchemy.ext.declarative import declarative_base

from sqlalchemy import create_engine

from application import init_db, db_session

init_db()

import application
test_app = application.app.test_client()

def teardown():
  db_session.remove()

{% endcodeblock %}

**testusers.py**

{% codeblock lang:python %}
import json
import nose
from nose.tools import *

from application import User

from tests import test_app

def check_content_type(headers):
  eq_(headers['Content-Type'], 'application/json')

def test_user_routes():
  rv = test_app.get('/users')
  check_content_type(rv.headers)
  resp = json.loads(rv.data)
  #make sure we get a response
  eq_(rv.status_code,200)
  #make sure there are no users
  eq_(len(resp), 0)

  #create a user
  d = dict(first_name="User1First", last_name="User1Last",email="User1@User1.com")
  rv = test_app.post('/users', data=d)
  check_content_type(rv.headers)
  eq_(rv.status_code,201)

  #Verify we sent the right data back
  resp = json.loads(rv.data)
  eq_(resp["email"],"User1@User1.com")
  eq_(resp["first_name"],"User1First")
  eq_(resp["last_name"],"User1Last")

  #Get users again...should have one
  rv = test_app.get('/users')
  check_content_type(rv.headers)
  resp = json.loads(rv.data)
  #make sure we get a response
  eq_(rv.status_code,200)
  eq_(len(resp), 1)

  #GET the user with specified ID
  rv = test_app.get('/users/%s' % resp[0]['id'])
  check_content_type(rv.headers)
  eq_(rv.status_code,200)
  resp = json.loads(rv.data)
  eq_(resp["email"],"User1@User1.com")
  eq_(resp["first_name"],"User1First")
  eq_(resp["last_name"],"User1Last")

  #Try and add Duplicate User Email
  rv = test_app.post('/users', data=d)
  check_content_type(rv.headers)
  eq_(rv.status_code,500)
{% endcodeblock %}

For the tests, I create a simple **runtests.sh** shell script, which is where I specify a different database to use when running tests.  It has the following line:

{% codeblock %}
DATABASE_URL=postgres://flaskexample:flask@localhost:5432/flaskexample_test nosetests --nocapture
{% endcodeblock %}

When running the **runtests.sh** file, you should get some output like:

{% codeblock console %}
 $ ./runtests.sh
.
----------------------------------------------------------------------
Ran 1 test in 0.277s

OK
{% endcodeblock %}

This means your tests have run successfully.  If you go check your **flaskexample_test** database, you should have one user in the **Users** table.  When you run the test again, that user will be removed via the init_db() call, and the tests will run against a blank database.

Obviously, this is a very simple example, but I think it is a good starting point for setting up Test Driven Development with Flask and SQLAlchemy.

Feel free to email me with any questions or comments.  The git repo for this is located [here](https://github.com/kelsmj/flask-test-example)
