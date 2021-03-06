# Express & Postgres

### Objectives
1. Learn what an ORM is and what purpose it serves in server side programs.
2. Learn how the semantic naming convention of REST urls for a resource type
   corresponds to a standard set of CRUD pages (index, new, edit, delete).
3. Install the Sequelize ORM into an Express app and use it to connect to and
   modify our application database.
4. Use the Sequelize CLI to create a `user` model and database migration.
5. Use our user model to CRUD user data that is recieved by our app in
   the our express routes.

### Definitions

- Object-Relational Mapping (ORM) - in computer science is a programming
  technique for converting data between incompatible type systems in
  object-oriented programming languages. This creates, in effect, a "virtual
  object database" that can be used from within the programming language. -
  [wikipedia](https://en.wikipedia.org/wiki/Object-relational_mapping)

- Migration - In software engineering, schema migration (also database
  migration, database change management) refers to the management of
  incremental, reversible changes to relational database schemas. -
  [wikipedia](https://en.wikipedia.org/wiki/Schema_migration)

- Model - The central component of MVC, the model, captures the application's
  behavior in terms of its problem domain, independent of the user interface.
  The model directly manages the application's data, logic and rules. So the
  Model is the biggest, most important layer in most MVC applications
  [stackoverflow](http://stackoverflow.com/questions/5093880/what-is-the-usage-of-model-in-mvc-is-it-actually-useful)

## Setup

Copy the `fazbook` directory to a location **outside** of your WDI_HAKUNA_MATATA
repository. We will be initializing a git project there so we can later deploy this to
Heroku. In order to do this, **the app direcotry cannot be inside of an existing
git project**. 

Once it is copied, `cd` into the copy and run `npm install`

Run `npm run dev` to make sure that everything is working properly. 

Run `git init` to create the git repository and make your first commit.

Go to GitHub and create a new repository *(leave the 'create with readme' box
UNCHECKED)* and follow the two lines of instructions for pushing up an existing
git repository.

## Sequelize - A Node ORM

Sequelize is a library that serves as an ORM for Node applications. It's primary
function is to act as a layer between the relational database of your
application and the code that constitutes your app server. We will be using
Sequelize to create database tables as well as insert, update, retrieve and
delete data from our database.

### Install the Sequelize CLI

First we are going to install the sequelize cli and pg. We will first do this globally so we have the
`sequelize` command available, then with the `--save` flag so
it's added as a dependency to our package.json. We're also installing some
utilities that will help Sequelize hook up to Postgres.
```
npm install -g sequelize-cli pg
npm install --save sequelize sequelize-cli pg pg-hstore
```

After that we need to add a `.sequelizerc` file to the root directory of our
app and fill it with:
```
var path = require('path');

module.exports = {
  'config': path.resolve('./server', 'config.json'),
  'migrations-path': path.resolve('./server', 'migrations'),
  'models-path': path.resolve('./server', 'models'),
  'seeders-path': path.resolve('./server', 'seeders')
}
```
Then run:
```
sequelize init
```

This will create a `server` director in your project root. Inside will be the
directories for your migrations, models, seeders and database configurations.

We now have to set up our development database. To do that run:
```
createdb fazbook_development
```

Last thing you have to do before Sequelize can work properly is modify your
configuration file to point to your development database. Go to your
`server/config.json` file and change it to look like this:
```
{
  "development": {
    "username": "YOUR_DB_USERNAME",
    "password": null,
    "database": "fazbook_development",
    "host": "127.0.0.1",
    "dialect": "postgres"
  }
}
```

## Create a user model

```
sequelize model:create --name User --attributes "email:string firstName:string lastName:string dob:date"
sequelize db:migrate
```
### GET /users/new

Now lets add the ability to create users.

In your `users/index.ejs` file, add a link to `/users/new` and label it
something like "Create a user".

Now add a get route for the `/new` path in your users routes file.

Now add a `new.ejs` template in your `views/users` folder. In there you should
add a form with `email`, `firstName` and `lastName` text inputs. Also be sure
to include a `dob` date type input. The form method should be `POST` and the
action should be `/users`.

What happens when you click the submit button?

### POST /users

First we have to require our Sequelize `models` object. On line 3 of the file
add:
```javascript
var models = require('../server/models/index');
```
This is needed now because we are actually communicating with our Postgres db.
(The new user page didn't need to).

We need to add another route to handle the POST request to the `/users`
endpoint.

```javascript
router.post('/', function(req, res, next) {
  models.User.create({
    firstName: req.body.firstName,
    lastName: req.body.lastName,
    email: req.body.email,
    dob: req.body.dob
  }).then(function() {
    res.redirect('/users')
  });
});
```

This uses the Sequelize model `create` method to add a user record to the
database. The object you provide tells it what attributes to set. After the user
record is committed to the db, the `then` callback redirects us to the users
index page.

After you submit you form and the POST request is sent to create your user, use
Postico to confirm that your record has been added (it won't work in psql for
some reason... 🤷🏽‍♂️)

### GET /users

Now that we have user records in our db we need to display them on our index
page. In our `routes/users.js` file we already have a route for the users index page.
We want to change this to fetch all of the users from our database and provide
them as JS objects for use in our ejs template.

Modify your `/` route to look like this:

```javascript
router.get('/', function(req, res, next) {
  models.User.findAll({}).then(function(users) {
    res.render('users/index', {
      title: 'fazbook',
      users: users
    });
  });
});
```

`findAll` is Sequelize model method that queries the database for all of the
records that match the provided options list. Our options list is empty so it
will not filter the results and will return everything in the `Users` table.

Now that we have the `users` array available in our `views/users/index.ejs`
file, add a `forEach` to iterate through the `users` array and render a div that
displays the email, first name, last name and dob of each user object.

```
git add -A
git commit -m "Add user create functionality and display users index page"
```

### DELETE /users/:id

Now we have to add a button next to each user on the users index page to delete
that user. To do this, add a form within each user div on the users index page.
The form method should be POST and the actions should be `/users/:id?_method=DELETE`.

This is a strange url syntax.The reason we write delete requests as POSTs like
this is because html forms cannot send delete requests. In order to
use delete requests in our app we will have to send the request as a POST and
include a special `_method` query parameter to indicate that we intend for it to
act as a delete request.

We also have to add a middleware that can convert these requests. We are going to
use a middleware called method-override for this.

```
npm install --save method-override
```

Then in your `app.js` add:
```
var methodOverride = require('method-override');
```
after you require express. (You can put it on line 7)

Then you have to use it. I would do this right after you set the `app` variable
on line 13.
```
// override with POST having ?_method=DELETE
app.use(methodOverride('_method'));
```

Now we can add the delete route.
```javascript
router.delete('/:id', function(req, res, next) {
  models.User.destroy({
    where: { id: req.params.id }
  }).then(function(user) {
    res.redirect('/users');
  });
});
```

```
git add -A
git commit -m "Add ability to delete users"
```

### GET /users/:id

Edit your `users/index.ejs` template to display a link for each user div. The
href should be `/users/id` where id is the id of the user object. (Use ejs for
the string concatenation to make this url for each user object in the forEach
loop). The link will display the users first name. Remove the other user
details.

Add a get route for `/users/:id`. In this route we will user the Sequelize
`findById` method to find the user.
```javascript
router.get('/:id', function(req, res, next) {
  models.User.findById(req.params.id).then(function(user) {
    res.render('users/show', { user: user });
  });
});
```

Create a `show.ejs` template in your `views/users` directory. This is where you
can display the rest of the user details that you removed from the index view.

```
git add -A
git commit -m "Add users show page"
```

### GET /users/:id/edit

Now we are going to create an edit link on the user show page that will bring
us to the user edit page. The link should have an href of `/users/id/edit` and
the edit route should use the sequelize `fineById` method to get the user based
on the id in the url. It will render the edit page for that user.
```javascript
router.get('/:id/edit', function(req, res, next) {
  models.User.findById(req.params.id).then(function(user) {
    res.render('users/edit', { user: user });
  });
});
```

Now create the `edit.ejs` template. This will be a form similar to the new user
form but the user object is now available for prepopulating the values of the
form inputs. The `<form>` tag attributes will be similar to the delete forms on the user
index page (because html forms cant send PUT requests either...)

### PUT /users/:id

Create a put route for `/users/id`. This route will use the sequelize `update`
command to update the record with the id provided in the url.
```javascript
router.put('/:id', function(req, res, next) {
  models.User.update({
    email: req.body.email,
    firstName: req.body.firstName,
    lastName: req.body.lastName,
    dob: req.body.dob
  }, { where: { id: req.params.id } }).then(function() {
    res.redirect('/users/' + req.params.id);
  });
});
```

Notice here that we're redirecting to the users show page after the update. This
will allow the user to confirm that the updates have been committed to the
database.

```
git add -A
git commit -m "Add ability to update users"
```

## Deploy
Using Heroku CLI: `heroku create`. If it doesn't push it up for some reason run
`git push heroku master`

**OR**

Create a new app in Heroku. Click on the Deploy tab and follow the instructions
to deploy an existing app (do not use the heroku create command). Push it up.

**THEN**

Go the the Heroku dashboard for you app and click on the Resources tab.

In the add-ons section type postgres and click Heroku Postgres. Leave the
default Hobby Dev - Free option selected and click the Provision button.

Modify your server/config.json to look like this:
```
{
  "development": {
    "username": "YOUR_DB_USERNAME",
    "password": null,
    "database": "fazbook_development",
    "host": "127.0.0.1",
    "dialect": "postgres"
  },
  "production": {
    "use_env_variable": "DATABASE_URL"
  }
}
```

Replace replace 28-30 in your `bin/www` file with this:
```
var models = require('../server/models/index');
models.sequelize.sync().then(function () {
  server.listen(port);
  server.on('error', onError);
  server.on('listening', onListening);
});
```

Then:
```
git add -A
git commit -m "Add production db configuration"
git push origin master
git push heroku master
```

### Solution
[code](https://github.com/parisyee/fazbook) | [app](https://rocky-fjord-23105.herokuapp.com/)
