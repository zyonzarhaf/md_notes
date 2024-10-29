# Notes on CakePHP

## The Entry Point of the App

The application entry point is webroot/index.php. When a request reaches the app URL, the web server (Apache or Nginx) routes the request to the index.php file. This is what the file looks like:

```php
<?php

require dirname(__DIR__) . '/config/requirements.php';

if (PHP_SAPI === 'cli-server') {
    $_SERVER['PHP_SELF'] = '/' . basename(__FILE__),;

    $url = parse_url(urldecode($_SERVER['REQUEST_URI']));
    $file = __DIR__ . $url['path'];
    if (strpos($url['path'], '..') === false && strpos($url['path'], '.') !== false && is_file($file)) {
        return false;
    }
}
require dirname(__DIR__) . '/vendor/autoload.php';

use App\Application;
use Cake\Http\Server;

$server = new Server(new Application(dirname(__DIR__) . '/config'));

$server->emit($server->run());

```

The file uses 'basename()', 'dirname()', __FILE__, __DIR__, an instance of the 'Server' class, and an instance of the 'Application' class to:

1. call the autoload routine from the full path to '../vendor/autoload.php', allowing the application to use all the dependencies defined in the composer.json file.

1. import the 'Application' class from the app namespace.

1. import the 'Server' class from the app namespace.

1. instantiate a server, while also passing in an application instance, which then takes the full path to '../config' directory.

1. call the appropriate 'Server' methods to instantiate the request (internally, cakephp uses the built-in global variables to achieve this) and the response objects.

## The Model Layer

The **Model layer** represents the part of the app that implements the business logic. It is responsible for: **retrieving data, converting data, validating data, associating data, and other tasks related to handling data.** 

Below is an example of fetching the 'Users' table **from within its corresponding controller**, by using the `fetchTable()` method (alternatively, we could use the default table configured for the controller: `$this->TableName`).

We use one of the methods provided by the 'Table' class -- the `find()` method --, which returns an instance of the SelectQuery class. This class exposes various methods to refine the query and always return itself (therefore it is chainable), unless one of these methods are called at the bottom of the chain:

- all()

- execute()

- first(),

- toList(),

- toArray()

Example code for building a query using `find()`:

```php

<?php

$query = $this->fetchTable('Users')
    ->find()
    ->select(['id', 'username']) // Same query
    // it's also possible to provide aliases to the selected fields: 
    // select(['pk' => 'id', 'aliased_title' => 'title', 'complete_name' = 'name']);
    ->where(['id !=' => 1]) // Same query
    ->orderBy(['created' => 'DESC']); // Same query

```

However, in order to iterate the result set, we **have** to execute the query. Example:

```php

<?php

use Cake\ORM\Locator\LocatorAwareTrait;

$query = $this->fetchTable('Users')
    ->find()
    ->select(['id', 'username']) // Same query
    ->where(['id !=' => 1]) // Same query
    ->orderBy(['created' => 'DESC']) // Same query
    ->all() // Execute the query

foreach($resultset as $row) {
    echo $row->username;
}

```

If, however, the query gets executed, several traversing methods will be available to use, such as map(), filter(), reduce(), count(), extract(), combine(). [And many others](https://book.cakephp.org/5/en/core-libraries/collections.html)

Another example would be creating a new user aka writting a new row to the database:

```php
<?php

$users = $this->fetchTable('Users');
$user = $users->newEntity(['email' => 'foo@bar.com']);
$users->save($user);

```

## The View Layer

The **View layer** renders a presentation of modeled data. By gaining access to an iterable object that represents a given dataset, we wrap the data in html elements:

```php
<?php

<?php foreach ($resultset as $row): ?>
<li class="user">
    <?= $this->element('user_info', ['user' => $user]) ?>
</li>
<? endforeach; ?>

```

The AppView is the default view class, and it can be used to load presentation helpers that will be used for every view rendered in the app. Example: 

```php

<?php

namespace App\View;

use Cake\View\View;

class AppView extends View
{
    public function initialize(): void
    {
        $this->addHelper('MyUtils');
    }
}

```

The View layer is not only used to render HTML for browser-based clients. It can also reply with JSON, CSV, XML, and PDF.

This layer consists of five main components: templates, elements, layouts, helpers, and cells. Templates are the part of the page that is unique to a given controller and action. Elements are reusable bits of view code. Layouts are presentational code that wraps other interfaces in the app. Helpers encapsulate view logic. Finally, cells provide mini controller-like features for creating self contained UI components.

As already mentioned, the template gains access to the data when its corresponding controller calls the 'set()' method with the desired data. However, templates also have a 'set()' method. By calling it, this layer passes the variables to the layout and to the elements that will be rendered later.

### View Blocks

One of the features provided by the 'AppView' class is the view block. A view block is a code block that can be defined somewhere in src/template/* and then imported somewhere else, typically somewhere in src/template/layout/*. 

A block can be defined either as a capturing block, or by direct assignment.

Example of a capturing block:

```php
<?php
// In src/template/Articles/index.php

$this->start('scriptBottom');
<script>
  $(function() {
    //Initialize Select2 Elements
    $('.search-select, .select2').select2({
      "language": "pt-BR"
    });

    $('.open-box-filter').on('click', function(e) {
      $('.box-filter').slideToggle();
    });
  });
</script>
$this->end(); 

// Somewhere in src/template/layout/

<?php
$this->fetch('scriptBottom');
?>

```

Example of a direct assignment:

```php
<?php
// In src/template/Articles/index.php
$this->assign('title', 'Dashboard');

// Somewhere in src/template/layout/
$this->fetch('title');

// Also, it's possible to set a default value, like:
$this->fetch('title', 'Default Title');

```
As demonstrated, capturing blocks are useful for multiple lines of code. Direct assignment, on the other hand, is more appropriate for one liners or simple values.

However, the fetching part is the same for both cases: you define a thing somewhere and fetch it somewhere else.

### Elements

another way to get data outside the template is by creating an element. elements are reusable pieces of presentation code that can be shared between different templates. they are located at src/template/element. you can build your own folder structure within this location to better organize the components. example: 

```php
<?php
// in src/template/element/someentity/common_select.ctp
$categorias = [
    'categoria 1' => 'categoria 1',
    'categoria 2' => 'categoria 2'
];

echo $this->form->control('categorias', [
    'label' => 'tipo:',
    'options' => $categorias,
    'empty' => '(selecione uma categoria)'
]);


// in src/template/someentity/add.ctp
<?php echo $this->form->create(); ?>
<div>
    <div>
        <?php
        echo $this->element('someentity/common_select');
        ?>
    </div>
</div>
<!-- /.box-body -->
<?php echo $this->form->submit(__('add category')); ?>
<?php echo $this->form->end(); ?>
</div>

```

### helpers

#### form

the formhelper is used to create html forms. it automatically renders a given set of input controllers through context.

the context is the first parameter it takes. it can be a complete orm entity, orm resultset, form instance, array containing the 'schema key', array of metadatada, or null. validation will also be automatically applied.

the second parameter is used to define things like the http request method, the url to request from, encoding, enctype, among other options.

by default, the form makes a 'post' request to the same url it matches.

example of a simple form:

```php
<?php

<?= $this->form->create($department) ?>
<fieldset>
    <legend><?= __('add a department') ?></legend>
    <?php
    echo $this->form->control('department');
    ?>
</fieldset>
<?= $this->form->button(__('submit')) ?>
<?= $this->form->end() ?>

```

#### Html

the html helper is a more general-purpose helper that helps creating html elements. [todo].

This helper also ties into view blocks by providing the 'script()', 'css()', and 'meta()' methods, which take the basename of a file and an array of options. The array of options takes a 'block' key, which holds the value describing the name of the block that will be captured somewhere else. Example:

```php
<?php

// in src/template/Articles/index.php
$this->Html->script('carousel', ['block' => 'scriptBottom']);

// Somewhere in src/template/layout
$this->fetch('scriptBottom');

```

CakePHP will automatically look for the file in webroot/js/carousel.js.

## the controller layer

the **controller layer** handles http requests and send http responses, but first it **delegates tasks to the other two layers**. it waits for petitions from clients, checks their validity according to authentication or authorization rules, delegates data fetching or processing to the model, selects the type of presentation data that the clients are accepting, and finally delegates the rendering process to the view layer. 

this is what a controller typically looks like:

```php
<?php

class userscontroller extends appcontroller
{
    public function initialize(): void
    {
        parent::initialize();

        // setup components that are needed across multiple actions
        // configure middlweare specific to the controller
        // setup logic that should run for every action in the controller
        // lifecycle management: initializing state

    }

    public function index()
    {

    }

    public function add()
    {

    }
}

```

some methods, such as 'index()', 'add()', 'edit()', among others, are automatically generated as boilerplate code by the bin/cake bake command. the 'initialize()' method can be very useful for initializing state and passing in common data and behavior to the controller actions, which in turn can pass data to the corresponding templates.

in the example below, the controller (literally, an instance of the 'controller' class) first access its default table through a magic get method. then, it calls the 'newemptyentity()' method to return an entityinterface object, which takes the data from the http post request through the 'patchentity()' method.

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

### Components

Components are reusable and plugable packages. They come in three flavors: built-in core components, downloadable and installable components, and custom components. Components main purpose is to share common logic between controllers, keeping them clean. They are usually loaded via the 'loadComponent()' controller method within a given controller:

```php
<?php

class PostsController extends AppController
{
    public function initialize()
    {
        parent::initialize();

        $this->loadComponent('FormProtection', [
            'unlockedActions' => ['index'],
        ]);

        $this->loadComponent('Flash');
    }
}

```

Alternatively, the same thing could be done through the 'beforeFilter()' controller method:

```php
<?php

public function beforeFilter(EventInterface $event)
{
    $this->FormProtection->setConfig('unlockedActions', ['index']);
}

```

The main difference between these two is the component timing of execution: the first approach yields in the component being loaded before any action is executed, and for every request to the controller; the second approach allows to restrict the execution of a given component to a specific action.

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

## CakePHP Naming Conventions

CakePHP follows the principle "convention over configuration". To follow the framework's conventions, all controller class names must be kept plural, pascal cased, and end with 'Controller', e.g. 'UsersController', 'MenuLinksController'.

Public methods within a given controller are exposed as 'actions' and match the same name as the routes exposed to the web browser. Actions must be kept camel cased, e.g. '/users/view-me' maps to 'UsersController'/'viewMe'.

For multiple word controller, the convention for their matching URL names is to keep everything lower cased and dashed, e.g. '/menu-links/view-all' maps to 'MenuLinksController'/viewAll.

### File and Class Name Conventions

Not much to talk about here. Filenames match the class names, e.g. ArticlesController.php and ArticlesController.

### Database Conventions

Table names corresponding to CakePHP models must be kept plural and snake cased, e.g. users, menu_links, and user_favorite_pages, would map to Users, MenuLinks, and UserFavoritePages.

Columns with more than one word are also snake cased, but they can be kept in the singular, e.g. first_name.

**Foreign keys** in 'hasMany', 'belongsTo', and 'hasOne' relationships are recognized **by default** as the singular name of the related table followed by '_id'. For example:if a Users table 'hasMany' Articles -- in a 'hasMany' relationship, the fk is in the related table --, the Articles table will refer to the User table via 'user_id'.

For 'belongsToMany' relationships, which describe associative tables, the associative table must be named after the tables it connects, with both names pluralized and sorted alphabetically, e.g. articles_tags. **To avoid conflicts** the other tables should not contain related foreign keys, **only the associative table**. Also to avoid conflicts, the associative table must contain a single primary key, typically one of the foreing keys.

### Model Conventions

Table class names are plural, pascal cased and end in 'Table', e.g. 'UsersTable', 'MenuLinksTable', and 'UserFavoritePagesTable'.

Entity class names, on the other hand, are singular, pascal cased and have no suffix, e.g. 'User', 'MenuLink', 'UserFavoritePage'.

### View Conventions

View templates files are named after the controller functions they display, but are all kept snake cased, e.g. ArticlesController::viewAll() will work with the view template templates/Articles/view_all.php.

## Authentication

Authentication is the process of verifying a user's identity. According to the official docs, this is the typica authentication workflow:

1. $ composer require cakephp/authentication.

1. Load the authentication plugin inside src/Application.php.

1. Add the class imports.

1. Add AuthenticationServiceProviderInterface to the implemented interfaces on the app entry point class.

1. Add a new middleware to the middleware() method.

1. The new middleware will call a hook method that allows the app to define a specific auth service. Define the hook method.

1. Add the authentication component in src/AppController.php

1. Optionally allow specific unauthenticated rotes in specific controllers.

1. Add login and logout actions to src/Controllers/UsersController.php

1. Add templates.

## Authorization

Authorization is the process of defining who is allowed to access what.

## Access Control List (ACL) -- Older versions only

### Definition

Access Control Lists are a more granular approach to authorization. "The ACL system allows developers to define complex permission structures, enabling fine-tuned control over who can access what within an application. Permissions are typically stored in a database and can be managed through CakePHP's built-in models and console commands.

ACLs consist of two main components: 'acos' (Access Control Objects) and 'aros' (Access Request Objects). Both 'acos' and 'aros' objects are stored in the database, and they are linked through an 'aros_acos' joint table. This is what they look like:

```sql

CREATE TABLE `acos` (
`id` int(11) NOT NULL,
`parent_id` int(11) DEFAULT NULL,
`model` varchar(255) DEFAULT NULL,
`foreign_key` int(11) DEFAULT NULL,
`alias` varchar(255) DEFAULT NULL,
`lft` int(11) DEFAULT NULL,
`rght` int(11) DEFAULT NULL
)


CREATE TABLE `aros` (
`id` int(11) NOT NULL,
`parent_id` int(11) DEFAULT NULL,
`model` varchar(255) DEFAULT NULL,
`foreign_key` int(11) DEFAULT NULL,
`alias` varchar(255) DEFAULT NULL,
`lft` int(11) DEFAULT NULL,
`rght` int(11) DEFAULT NULL
)

CREATE TABLE `aros_acos` (
`id` int(11) NOT NULL,
`aro_id` int(11) NOT NULL,
`aco_id` int(11) NOT NULL,
`_create` varchar(2) NOT NULL DEFAULT '0',
`_read` varchar(2) NOT NULL DEFAULT '0',
`_update` varchar(2) NOT NULL DEFAULT '0',
`_delete` varchar(2) NOT NULL DEFAULT '0'
)

```

The 'id' is the primary key used to identify a specific row in the 'acos' or 'aros' table. The 'parent_id' references the parent ACO, establishing a hierarchy; 'alias' is the name of the resource (e.g., controller or action). The 'aros_acos' table is an associative table that links both 'acos' and 'aros' in a many to many relationship.

### Creating ACOs and AROs

These tables can be initially created through `bin/cake Migrations.migrations migrate -p Acl`.

Now, for specific rows within the 'aros' table, the Acl plugin goes through each table within src/Model and looks for an added behavior (requester), like this:

```php
<?php

public function initialize(array $config)
{
    parent::initialize($config);

    // Other method calls

    $this->addBehavior('Acl.Acl', ['type' => 'requester']);

    // Table relationships
}

```

For specific rows within the 'acos' table, the plugin simply scans the whole src/Controller folder and create an entry for each specific action and controller.

The entries are generated through the `bin/cake acl_extras aco_sync` command.

### Configuring Permissions

Permissions are granted or denied through shell commands: 

```sh

bin/cake acl grant Groups.1 controllers
bin/cake acl deny  Groups.2  controllers
bin/cake acl grant Groups.2 controllers/Posts

```

In this example, members of the group id 1 were grant the permission to everything; members of the group id 2 were denied everything; members of the same group were grant the permission to every action in controllers/Posts.

Each command will yield in a new entry in the 'aros_acos' table. Then, inside the app, permissions can be checked through the 'check()' method.

## Search

Creates paginate-able filters for a CakePHP application. Install through composer and then run `bin/cake plugin load Search` to load the plugin in src/Application.php. After that:

1. Attach the Search behavior to the desired table class:

```php
<?php

public function initialize(array $config): void
{
    parent::initialize($config);

    $this->addBehavior('Search.Search');
}

```

1. Then, call the 'searchManager()' method to return a search manager instance, and chain all the desired filters in it through the add() method. This method takes the name of the field to search for, the name of the filter to be used (using the dot notation, e.g. 'Search.Like'), and an associative array of options to the specific filter. All filters support [the following options](https://github.com/FriendsOfCake/search/blob/master/docs/filters-and-examples.md#options).

```php
<?php

public function initialize(array $config): void
{
    parent::initialize($config);

    $this->addBehavior('Search.Search');
    $this->searchManager()
        ->add('fieldName', 'Filter.Name', [
            // Array of filter specific options
        ])
        ->add()
        ->add()
        ->add()
        ->add()
}

```

## CakePHP AdminLTE Theme

CakePHP plugin to integrate the AdminLTE theme into the application, compatible with CakePHP 4.X. It provides a clean an responsive admin interface, leveraging the popular AdminLTE templates.

How to use it:

1. Require the plugin via composer.

1. Enable the plugin in src/Application.php.

1. Set the theme in src/Controller/AppController.php.

The typical workflow to integrate the theme into the application is to simply generate all the necessary files through normal `bin/cake bake [whatever you want to bake] [table name]` commands, plus a `--theme AdminLTE`at the end.

## Console Commands

CakePHP comes with an executable file in the bin directory called 'cake'. This file is used to run console commands. To list out all available commands you run `bin/cake` with no arguments. Making the file executable and/or double-checking the database connection are common solutions when troubleshooting the bake command.

After listing out the available commands, usually running `bin/cake [command name] --help` will print a list of the arguments and options a given command expects.

Cake automatically detects commands through a dispatcher-type system that looks for whatever is in 'src/Command/'.

### Bake Console

A huge part of CakePHP console commands is the bake command. Bake can create any of the CakePHP's basic ingredients, and is specially handy to cook MVC skeletons, by simply running: `bin/cake bake [whatever you want to bake] [table name] [options]`.

In CakePHP, you can overwrite the default output of the bake command for a given "ingredient" by using the option '--theme [name of the theme]' at the end of a bake command.

## Debugging

Debugging is typically done through the 'debug()' and the 'dd()' methods, as long as the 'DEBUG' environment variable is set to true in config/app.php:

```php
<?php

return [
    /*
     * Debug Level:
     *
     * Production Mode:
     * false: No error messages, errors, or warnings shown.
     *
     * Development Mode:
     * true: Errors and warnings shown.
     */
    'debug' => filter_var(env('DEBUG', true), FILTER_VALIDATE_BOOLEAN),
// Rest of the code
];

```

By simply calling 'debug()' and passing a variable, CakePHP will be able to render both the template and whatever is in the variable you passed. On the other hand, 'dd()'will render what's in the variable and die, meaning the control flow will simply stop at the line where the method first got called.

```php
<?php

$configemailstatus = [
    [
        "titulomensagem" => '',
        "descricao" => '',
        "statuscontrato" => '',
        "distribuidor_id" => 0,
        "ativo" => false,
        "motivo" => ''
    ],
    [
        "titulomensagem" => '',
        "descricao" => '',
        "statuscontrato" => '',
        "distribuidor_id" => 0,
        "ativo" => false,
        "motivo" => ''
    ],
    [
        "titulomensagem" => '',
        "descricao" => '',
        "statuscontrato" => '',
        "distribuidor_id" => 0,
        "ativo" => false,
        "motivo" => ''
    ]
];

$configemailstatusCopia = array_reduce($configemailstatus, function($initial, $item) {

}, []);

```
