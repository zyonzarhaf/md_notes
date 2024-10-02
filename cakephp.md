# Notes on CakePHP

## CakePHP at a Glance

### The Model Layer

The **Model layer** represents the part of the app that implements the business logic. It is responsible for: **retrieving data, converting data data, validating data, associating data, and other tasks related to handling data.** 

Below is an example of fetching the 'Users' table with the 'fetchTable()' method, and using one of the methods provided by the 'Table' class -- the 'find()' method -- we can also return an instance of the 'Query' class. 

By also calling 'all()', we execute the query and effectively retrieve all rows from the 'Users' table as a traversable object -- a 'ResultSetDecorator' object aka 'QueryInterface'. The QueryInterface allows processing data by means of iterating and also by exposing its own methods.

In this example, we simply iterate the collection and call the echo command for each username in a row.

```php
<?php

use Cake\ORM\Locator\LocatorAwareTrait;

$users = $this->fetchTable('Users');
$resultset = $users->find()->all();

foreach($resultset as $row) {
    echo $row->username;
}

```

Another example would be creating a new user aka writting a new row to the database:

```php
<?php

$users = $this->fetchTable('Users');
$user = $users->newEntity(['email' => 'foo@bar.com']);
$users->save($user);

```

### The View Layer

The **View layer** renders a presentation of modeled data. By gaining access to an iterable object that represents a given dataset, we wrap the data in html elements:

```php
<?php

<?php foreach ($resultset as $row): ?>
    <li class="user">
        <?= $this->element('user_info', ['user' => $user]) ?>
    </li>
<? endforeach; ?>

```

### The Controller layer

The **Controller layer** handles http requests and send http responses, but first it **delegates tasks to the other two layers**. It waits for petitions from clients, checks their validity according to authentication or authorization rules, delegates data fetching or processing to the model, selects the type of presentation data that the clients are accepting, and finally delegates the rendering process to the View layer. 

In the example below, the controller (literally, an instance of the 'Controller' class) first access its default table through a magic get method. Then, it calls the 'newEmptyEntity()' method to return an EntityInterface object, which takes the data from the http post request through the 'patchEntity()' method.

It then calls the 'save()' method from the Table object, by passing the recently created row. If 'save()' returns true, then all went ok and the 'Users' table now includes a new user.

Finally it sets a few flash messages, as well as the new user to the View layer, and automatically calls 'render()'.

```php
<?php

public function add()
{
    $user = $this->Users->newEmptyEntity();
    if ($this->request->is('post')) {
        $user = $this->Users->patchEntity($user, $this->request->getData());
        if ($this->Users->save($user, ['validate' => 'registration'])) {
            $this->Flash->success(__('You are now registered.'));
        }
        else {
            $this->Flash->error(__('Failed to register.'));
        }
    }
    $this->set(compact('user'));
}

```

### The Request Cycle

This is how the request cycle works in CakePHP:

1. The user requests a page or resource in our application.

1. The webserver rewrite rules direct the request to webroot/index.php.

1. Your Application is loaded and bound to an HttpServer.

1. Your application’s middleware is initialized.

1. A request and response is dispatched through the PSR-7 Middleware that your application uses. Typically this includes error trapping and routing.

1. If no response is returned from the middleware and the request contains routing information, a controller & action are selected.

1. The controller’s action is called and the controller interacts with the required Models and Components.

1. The controller delegates response creation to the View to generate the output resulting from the model data.

1. The view uses Helpers and Cells to generate the response body and headers.

1. The response is sent back out through the /controllers/middleware.

1. The HttpServer emits the response to the webserver.

### CakePHP Conventions

CakePHP follows the principle "convention over configuration". To follow the framework's conventions, all controller class names must be kept plural, pascal cased, and end with 'Controller', e.g. 'UsersController', 'MenuLinksController'.

Public methods within a given controller are exposed as 'actions' and match the same name as the routes exposed to the web browser. Actions must be kept camel cased, e.g. '/users/view-me' maps to 'UsersController'/'viewMe'.

For multiple word controller, the convention for their matching URL names is to keep everything lower cased and dashed, e.g. '/menu-links/view-all' maps to 'MenuLinksController'/viewAll.

### File and Class Name Conventions

Not much to talk about here. Filenames match the class names, e.g. ArticlesController.php and ArticlesController.

### Database Conventions

Table names corresponding to CakePHP models must be kept plural and snake cased, e.g. users, menu_links, and user_favorite_pages, would map to Users, MenuLinks, and UserFavoritePages.

Columns with more than one word are also snake cased, but they can be kept in the singular, e.g. first_name.

**Foreign keys** in 'hasMany', 'belongsTo', and 'hasOne' relationships are recognized **by default** as the singular name of the related table followed by '_id'. For example:if a Users table 'hasMany' Articles -- in a 'hasMany' relationship, the fk is in the related table --, the Articles table will refer to the User table via 'user_id'.

For 'belongsToMany' relationships, which describe associative tables, the associative table must be named after the tables it connects, with both names pluralized and sorted alphabetically, e.g. articles_tags.

### Model Conventions

Table class names are plural, pascal cased and end in 'Table', e.g. 'UsersTable', 'MenuLinksTable', and 'UserFavoritePagesTable'.

Entity class names, on the other hand, are singular, pascal cased and have no suffix, e.g. 'User', 'MenuLink', 'UserFavoritePage'.

### View Conventions

View templates files are named after the controller functions they display, but are all kept snake cased, e.g. ArticlesController::viewAll() will work with the view template templates/Articles/view_all.php.
