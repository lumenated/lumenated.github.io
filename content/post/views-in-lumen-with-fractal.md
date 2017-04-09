+++
date = "2017-04-08T23:17:28+02:00"
title = "views in lumen"
draft = false
+++

We love the eloquent ORM which is included in the Lumen framework.
What we dislike is using eloquent to serialize our models and output them directly to the user.
We have been playing around with the fractal library and believe that we have come up with a nice design pattern which we would love to share with you. Before we introduce views we should evaluate why eloquent comes short in advanced serialization.

# Why not serialize eloquent models?

Lumen offers excellent support for basic serializing of Eloquent models. We can simply return an eloquent collection and lumen makes sure the output will be valid JSON in the response.

```php
class Controller {
    public function get($id) {
        return Model::first($id);
    }
}
```

This convenience becomes impracticable when there's a need to hide certain properties from our response.
Eloquent provides a build in solution with the _$hidden_ or _$visible_ property on a model.

```php
class MyModel extends Model {
    // This won't serialize the id property
    protected $hidden = ['id'];
}
```

It becomes more complicated when we want to change the format of a `Carbon` instance in the response.
We could use the _getAttribute_ method to achieve this but then we lose all functionality that `Carbon` has to offer.

```php
class MyModel extends Model {
    public function getSomeDateAttribute() {
        return $this->someDate->toISO8601String();
    }
}

// Returns an ISO8601 string which is NOT a carbon instance anymore
(new MyModel)->someDate;
```

# Using fractal
Fractal wants us to create a transformer which accepts an object (an eloquent model) and outputs an array.
This will be useful for serialization later on.

```php
class MyModelTransformer {
    public function transform(MyModel $model) {
        return [
            'name' => $model->name,
            'createdAt' => $model->created_at->toISO8601String(),
        ];
    }
}
```

To serialize our model we can use the transformer and wrap it inside a fractal item.

```php
class Controller {
    public function get($id) {
        // This is the fractal manager which does all the serialization
        $manager = new Manager();
        // We use Item to transform one Model instance
        $item = new Item(MyModel::first(1), new MyModelTransformer);
        // If we need to transform multiple items then we can use the Collection type.
        // It allows us to use the same transformer again.
        // So we define what properties should be exposed in one place only
        // items = new Collection(MyModel::all(). new MyModelTransformer);

        return response()->json($manager->createData($item)->toArray());
    }
}
```

At lumenated we like this pattern a lot but it has its shortcomings.
The biggest being the boilerplate. For every controller method that we are going to create we need to build either an item or collection. Then we should pass it to the _createData_ method and cast the result to an array. The solution we came up with is [fractal views] which abstracts the serialization away and only expects us to create the transformers.

```php
class MyModelView extends Lumenated\FractalViews\View
{
  // The fractal transformer that has to be used for this view
  protected $transformerClass = MyModelTransforme::class;
}
```

This view will expose:
- the _render_ method for serializing either an item or a collection based on the given arguments
- the _renderOne_ method which always serializes a single item
- the _renderMany_ method which always serializes a collection

If we replace the boilerplate with a view. Our code will look like this:

```php
class Controller {
    public function get($id) {
        $view = new MyModelView();
        return response()->json($view->render(Model::find(1));
    }
}
```

That's all there is to it. For more information and advanced usage please refer to the README on the [fractal views] repository.

# Contribute
We would love to grow and you can help us with that. Head over to the [fractal views] repository and leave a star.

[fractal views]: https://github.com/lumenated/fractal-views