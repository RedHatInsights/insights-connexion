Insights Connexion
===========================================

This is a python package intended to be used as a base for an Insights Platform REST API application. It does all the boilerplate required to use a Postgres database with Connexion/Flask API. 

Installation
--------------------
```
pipenv install -e "git+git://github.com/RedHatInsights/insights_connexion.git#egg=insights_connexion" -d
pipenv install -e "git+git://github.com/fijshion/connexion.git@body_middleware_fix#egg=connexion"
```

*Note*: The second package is temporary while a [bug fix PR](https://github.com/zalando/connexion/pull/738) is pending. After the bug fix is merged into connexion/master, connexion will be installed from zalando's master.

Required Project Structure
--------------------
- alembic.ini
- api/
- app.py
- config.ini
- db/
  - versions/
  - models.py
- oatts.values.json
- swagger/ 
  - api.spec.yaml
- test.py

**alembic.ini**
This is the ini to configure [alembic migrations](https://alembic.zzzcomputing.com/en/latest/tutorial.html#the-migration-environment). The following example will work out of the box:

```
[alembic]
script_location = insights_connexion:db/migrations
version_locations = ./db/versions

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

**app.py**
```
import insights_connexion.app as app
app.start()
```

**swagger/api.spec.yaml**

This is the Swagger spec for the REST API. It will be used by [Connexion](https://github.com/zalando/connexion/) to automatically serve and validate the routes.

**test.py**

This is where you will define your tests. The following is an example of how to use this package's OATTS test runner. It includes a custom seed() function to seed the database before the OATTS suite is executed.

```
from db.models import Tag
import factory
import insights_connexion.test.oatts as oatts


class TagFactory(factory.Factory):
    class Meta:
        model = Tag

    id = 'default'


async def seed():
    return (await TagFactory().create())


oatts.seed = seed
oatts.test()
```


**oatts.values.json**

This is passed to oatts --customValuesFile. See [oatts](https://github.com/google/oatts) for details.

**db/models.py**

This is where the [Gino](https://python-gino.readthedocs.io/en/latest/) models are defined. They can be defined anywhere, this location is just an example. The important piece is to use the base class from this package. Here's an example how to define them:

```
from insights_connexion.db.gino import db


class Tag(db.Model):
    __tablename__ = 'tags'

    id = db.Column(db.Unicode, primary_key=True)
```

**db/versions**

This is where the database migration scripts live. See the [alembic doc](https://alembic.zzzcomputing.com/en/latest/tutorial.html#create-a-migration-script) for more info on generating migration scripts.

**api/**

This is the directory Connexion will look in for the endpoint handling functions. Each endpoint needs a separate file. See the [Connexion routing docs](https://connexion.readthedocs.io/en/latest/routing.html) for details. You can access the SQL Alchemy session thru this package, e.g.

```
from db.models import Tag
from insights_connexion import responses

async def post(request=None):
    body = await request.json()
    tag_to_create = Tag(**body)
    created_tag = await tag_to_create.create()
    return responses.create(created_tag.dump())
```

**config.ini**

This contains all the application's configuration parameters. Each environment gets a separate section, e.g. dev, qa, prod. It requires at least the following parameters in the [DEFAULT] section (substituting the values for your app):
```
[DEFAULT]
db_name = tagservice
db_user = tagservice
db_password = tagservice
db_host = localhost
db_port = 5746
log_level = INFO
debug = True
port = 8080
db_pool_size = 30
db_max_overflow = 100
```

Each specific environment can override the defaults in a separate section, e.g.
```
[test]
db_name = tagservicetest
port = 8081

[qa]
db_password = qadbpassword
```

Migrating the Database
--------------------
`pipenv run alembic upgrade head`

Running The App
--------------------
`pipenv run python app.py`

Running The Tests
--------------------
`pipenv run python test.py`
