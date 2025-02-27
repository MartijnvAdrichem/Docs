# CRUD Tutorial

---

What's the simplest entity you can think of? It will probably be something like ```Tag```, which only holds an ```id``` and a ```name```. Let's create this new entity in the database, the model for it, then create a CRUD Panel to let admins manage entries for this entity.

We assume:
- you've [installed Backpack](/docs/{{version}}/installation);
- you don't already have a ```Tag``` model in your project;

<a name="generate-files"></a>
## Generate Files

Since we don't have an Eloquent model for it already, we're going to use [Jeffrey Way's Generators](https://github.com/laracasts/Laravel-5-Generators-Extended) package, which you've most likely installed along with Backpack, to generate the migration. 

```zsh
# STEP 0. create migration
php artisan make:migration:schema create_tags_table --model=0 --schema="name:string:unique"
php artisan migrate
```

Now that we have the ```tags``` table in the database, let's generate the actual files we'll be using:

```zsh
php artisan backpack:crud tag #use singular, not plural
```

The code above will have generated:
- a migration (```database/migrations/yyyy_mm_dd_xyz_create_tags_table.php```);
- a database table (```tags``` with just two columns: ```id``` and ```name```);
- a model (```app/Models/Tag.php```);
- a controller (```app/Http/Controllers/Admin/TagCrudController.php```);
- a request (```app/Http/Requests/TagCrudRequest.php```);
- a resource route, as a line inside ```routes/backpack/custom.php```;
- a new item in the sidebar menu, in ```resources/views/vendor/backpack/base/inc/sidebar_content.blade.php```;

**Next up:** we'll need to go through the generated files, and customize for our needs.

<a name="customize-generated-files"></a>
## Customize Generated Files

We'll skip the migration and database table, since there's nothing there specific to Backpack, nothing to customize, and we've already run the migration.

<a name="the-model"></a>
### The Model

Let's take a look at the generated model:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Backpack\CRUD\CrudTrait;

class Tag extends Model {

  use CrudTrait;

  /*
  |--------------------------------------------------------------------------
  | GLOBAL VARIABLES
  |--------------------------------------------------------------------------
  */

  protected $table = 'tags';
  // protected $primaryKey = 'id';
  // public $timestamps = false;
  // protected $guarded = ['id'];
  protected $fillable = [];
  // protected $hidden = [];
  // protected $dates = [];

  /*
  |--------------------------------------------------------------------------
  | FUNCTIONS
  |--------------------------------------------------------------------------
  */

  /*
  |--------------------------------------------------------------------------
  | RELATIONS
  |--------------------------------------------------------------------------
  */

  /*
  |--------------------------------------------------------------------------
  | SCOPES
  |--------------------------------------------------------------------------
  */

  /*
  |--------------------------------------------------------------------------
  | ACCESORS
  |--------------------------------------------------------------------------
  */

  /*
  |--------------------------------------------------------------------------
  | MUTATORS
  |--------------------------------------------------------------------------
  */
}
```

As we can see, this is a pretty standard Eloquent model. The only differences:
- our new model uses a trait Backpack needs, ```CrudTrait```;
- we have some comments to help us separate Eloquent logic in our Models;

As with all generated classes, we can see it's not exactly ready for use. It needs a little configuration. We should make sure:
- ```$table``` contains the right table name (it does);
- ```$primaryKey``` is correct (we can see it's ```id``` in the database table so let's just uncomment the generated line);
- ```$fillable``` contains all the attributes we want the admin to be able to change (none are present, let's add ```name```);
- ```$timestamps``` is configured, so that Eloquent can take care of them automatically (since we do have those columns in the database table, let's uncomment it and set it to ```true```);
- relationships to other models are properly defined (no need for our ```Tag``` model);

This is all _standard procedure_ for new Laravel models - nothing Backpack-specific here. Here are all the changes we would make to this specific ```Tag``` model:

```diff
    protected $table = 'tags';
-   // protected $primaryKey = 'id';
+   protected $primaryKey = 'id';
-   // public $timestamps = false;
+   public $timestamps = true;
    // protected $guarded = ['id'];
-   protected $fillable = [];
+   protected $fillable = ['name'];
    // protected $hidden = [];
    // protected $dates = [];
```

We're now done configuring the model - because we didn't already have a valid Eloquent model to use for our CRUD Panel. If we _did have_ a working Eloquent model, we only needed to add ```use \Backpack\CRUD\CrudTrait;```

<a name="the-controller"></a>
### The Controller

Let's take a look at ```app/Http/Controllers/Admin/TagCrudController.php```. It should look something like this:

```php
<?php namespace App\Http\Controllers\Admin;

use Backpack\CRUD\app\Http\Controllers\CrudController;
use App\Http\Requests\TagCrudRequest;

class TagCrudController extends CrudController {

  use \Backpack\CRUD\app\Http\Controllers\Operations\ListOperation;
  use \Backpack\CRUD\app\Http\Controllers\Operations\ShowOperation;
  use \Backpack\CRUD\app\Http\Controllers\Operations\CreateOperation;
  use \Backpack\CRUD\app\Http\Controllers\Operations\UpdateOperation;
  use \Backpack\CRUD\app\Http\Controllers\Operations\DeleteOperation;

  public function setup() 
  {
      $this->crud->setModel("App\Models\Tag");
      $this->crud->setRoute("admin/tag");
      $this->crud->setEntityNameStrings('tag', 'tags');

      // auto-magically guess fields & columns
      // it's heavily recommended that you replace this with actual addField and addColumn statements
      // inside either $this->crud->operation() closures, or inside setupXxxxOperation() methods
      $this->crud->setFromDb();
  }
}
```

What we should notice inside this TagCrudController is that:
- ```TagCrudController extends CrudController```;
- ```TagCrudController``` has a ```setup()``` method, where can configure how the CRUD panel works;
- All operations are enabled by using that operation's trait on the controller;
- Each operation is set up inside a ```setupXxxOperation()``` method;

Let's move our attention to the ```setup()``` method, which is the gateway to configuring our CRUD Panel.

As we can tell from the comments there, in most cases we _shouldn't_ use ```$this->crud->setFromDb()```, which automagically figures out which columns and fields to show. That's because for real models, in real projects, it would _never_ be able to 100% figure out which field types to use. Real projects are very custom - that's a fact. In real projects, models are complicated, use a bunch of fields types and you'll want to customize things. Instead of using ```setFromDb()``` then gradually changing what you don't like, **we heavily recommend you manually define all fields and columns you need**.

Since our ```Tag``` model is so simple, we _can_ leave it like this - it will work perfectly, since we only need a ```text``` field and a ```text``` column. But let's not do that. Let's define our fields and columns manually, like big boys:

```php
    public function setup()
    {
        $this->crud->setModel('App\Models\Tag');
        $this->crud->setRoute(config('backpack.base.route_prefix') . '/tag');
        $this->crud->setEntityNameStrings('tag', 'tags');

        $this->crud->operation('list', function() {
          $this->crud->addColumn(['name' => 'name', 'type' => 'text', 'label' => 'Name']);
        });

        $this->crud->operation(['create', 'update'], function() {
          $this->crud->addValidation(TagCrudRequest::class);
          $this->crud->addField(['name' => 'name', 'type' => 'text', 'label' => 'Name']);
        });
    }
}
```

This will:
- disable the ```setFromDb()``` functionality (since we deleted that line);
- add a simple ```text``` column for our ```name``` attribute to the ListEntries operation (the table view);
- add a simple ```text``` field for our ```name``` attribute to the Create and Update forms;

It the exact same thing ```setFromDb()``` would have figured out, but done manually. This way, if we want to add [other columns](/docs/{{version}}/crud-columns)) or [other fields](/docs/{{version}}/crud-fields), we can easily do that. If we want to change the label of the ```name``` field from ```Name``` to ```Tag name```, we just make that small change. The benefits of _not_ using ```setFromDb()``` will be more obvious once you use Backpack on real models, we promise.

Here, in the ```setup()``` method, is where you can also do a lot of other things, like enabling other operations, adding buttons, adding filters, customizing your query, etc. For a full list of the things you can do inside ```setup()``` check out our [cheat sheet](/docs/{{version}}/crud-cheat-sheet). 

But we can make this even better. For large CRUDs, you ```setup()``` method will quickly become very big. A different way to configure your operations is by creating methods that follow our ```setupXxxxOperation()``` naming convention. Similar to Laravel accessors and mutators, if you have a method with a name that looks like that in your Controller, we'll know that what you actually want is to define how that Xxxx operation works. So let's rewrite the above, using ```setupXxxxOperation()``` methods:
```php
    public function setup()
    {
        $this->crud->setModel('App\Models\Tag');
        $this->crud->setRoute(config('backpack.base.route_prefix') . '/tag');
        $this->crud->setEntityNameStrings('tag', 'tags');
    }

    protected function setupListOperation()
    {
        $this->crud->addColumn(['name' => 'name', 'type' => 'text', 'label' => 'Name']);
    }

    protected function setupCreateOperation()
    {
        $this->crud->addValidation(TagCrudRequest::class);
        $this->crud->addField(['name' => 'name', 'type' => 'text', 'label' => 'Name']);
    }

    protected function setupUpdateOperation()
    {
        $this->setupCreateOperation(); // if it's the same, why repeat it
    }
}
```

Much cleaner, don't you think? You can use both syntaxes, but for big CRUDs the second way of configuring operations will be more useful.

Next, let's continue to another generated file.

<a name="the-request"></a>
### The Request

Backpack will also generate a [standard FormRequest file](https://laravel.com/docs/master/validation#form-request-validation), that you can use for validation of the Create and Update forms. There is nothing Backpack-specific in here, but let's take a look at the generated ```app/Http/Requests/TagCrudRequest.php``` file:

```php
<?php

namespace App\Http\Requests;

use App\Http\Requests\Request;
use Illuminate\Foundation\Http\FormRequest;

class TagRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     *
     * @return bool
     */
    public function authorize()
    {
        // only allow updates if the user is logged in
        return backpack_auth()->check();
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            // 'name' => 'required|min:5|max:255'
        ];
    }

    /**
     * Get the validation attributes that apply to the request.
     *
     * @return array
     */
    public function attributes()
    {
        return [
            //
        ];
    }

    /**
     * Get the validation messages that apply to the request.
     *
     * @return array
     */
    public function messages()
    {
        return [
            //
        ];
    }
}

```

This file is a 100% pure FormRequest file - all Laravel, nothing particular to Backpack. In generated FormRequest files, no validation rules are imposed by default. But we do want ```name``` to be ```required``` and ```unique```, so let's do that, using the [standard Laravel validation rules](https://laravel.com/docs/master/validation#available-validation-rules):

```diff
    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
-            // 'name' => 'required|min:5|max:255'
+            'name' => 'required|min:5|max:255|unique:tags,name'
        ];
    }
```

> If your validation needs to be different between the Create and Update operations, [you can easily do that too](/docs/{{version}}/crud-operation-create#separate-requests-for-create-and-update), by specifying different FormRequest files for each operation.

<a name="the-route"></a>
### The Route

We have already generated our CRUD route, and we don't need to do anything about it, but let's check our ```routes/backpack/custom.php```. It should look like this:

```php
<?php

// --------------------------
// Custom Backpack Routes
// --------------------------
// This route file is loaded automatically by Backpack\Base.
// Routes you generate using Backpack\Generators will be placed here.

Route::group([
    'prefix'     => config('backpack.base.route_prefix', 'admin'),
    'middleware' => ['web', config('backpack.base.middleware_key', 'admin')],
    'namespace'  => 'App\Http\Controllers\Admin',
], function () { // custom admin routes
    // CRUD resources and other admin routes
    Route::crud('tag', 'Admin\TagCrudController');
}); // this should be the absolute last line of this file
```

Here, we can see that our routes have been placed:
- under a prefix that we can change in ```config/backpack/base.php```;
- under a middleware we can change in ```config/backpack/base.php```;
- inside the ```App\Http\Controllers\Admin``` namespace, because that's where our custom CrudControllers will be generated;

**It's generally a good idea to have the all admin routes in this separate file.** If you edit this file in the future, make sure you leave the last line intact, so that other routes can be automatically generated inside this file. And of course, add your routes inside this route group, so that:
- you have a single prefix for your admin routes (ex: ```admin/tag```, ```admin/product```, ```admin/dashboard```);
- all your admin panel functionality is protected by the same middleware;
- all your admin panel controllers live in one place (```App\Http\Controllers\Admin```);

<a name="the-menu-item"></a>
### The Menu Item

We've previously generated a menu item in the ```views/vendor/backpack/base/inc/sidebar_content.php``` file. You'll see this file is pure HTML. This will allow you to customize the menu as much as you want. The "active" state of the menu is done with JavaScript, based on the ```href``` attribute.

This is the bit that has been generated for you:

```php
<li class='nav-item'><a class='nav-link' href='{{ backpack_url('tag') }}'><i class='nav-icon fa fa-tag'></i> Tags</a></li>
```

You can of course change anything here, if you want.

<a name="the-result"></a>
## The result

You are now ready to go to ```your-app-name.domain/admin/tag``` and see your fully functional admin panel for ```Tags```. 

**Congratulations, you should now have a good understanding of how Backpack\CRUD works!** This is a very very basic example, but the process will be identical for Models with 50+ attributes, complicated logic, etc.
