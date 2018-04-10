# mongoose-rest-api

A generator for REST API business logic for use with Express.js. Given a Mongoose model and a unique ID name, mongoose-rest-api will generate a set of methods that can be used directly with Express.js routes. Separating the logic into distinct methods in this way exposes a clean, testable surface, independent of the Express http request handling. In addition, this gives you a nice way to keep your library of business logic DRY by reusing and expanding upon the individual methods.

Each method returns a promise which resolves to the result of the request. Once these endpoint functions are linked up to an Express.js route, the result will be automatically generated routes like the following examples:

- `GET /widgets` Return a list of widgets
- `GET /widgets/WIDGET_ID` Return a specific widget, given it's ID
- `GET /widgets?color=blue` Return a list of widgets with the `color` attribute `'blue'`
- `GET /widgets?color=!blue` Returns a list of widgets that have any color but blue
- `GET /widgets?sort=-age` Returns a list of widgets sorted by their `age` attribute in reverse order
- `POST /widgets` Given a request body, will create a new Widget and respond with it
- `PUT /widgets/WIDGET_ID` Given a request body will modify the Widget specified by WIDGET_ID and respond with the modified version
- `DELETE /widget/WIDGET_ID` Deletes the widget specified by WIDGET_ID and respond with the object as it was before its deletion

## Installation

`npm install mongoose-rest-api --save`

## Generated Endpoints

|Endpoint Name|Description|Promise Resolution|
|-------------|-----------|------------------|
|get|Respond with specified resource, or list of resources, unmodified.|If the request includes a resource identifier, the promise will resolve to the specified resource, otherwise it will resolve to a list of resources.|
|post|Create a new resource from the Request body|Promise resolves the newly created resource|
|put, patch|Modifies a specified resource|Promise resolves to the modified resource|
|delete|Deletes a specified resource|Promise resolves to the deleted resource as it was before the deletion|

## Accepted req.query parameters

### Get with ID

|Parameter|Type|Description|
|---------|----|-----------|
|columns|Array|A list of columns to include on resource. By default, all visible columns will be included|
|populate|Array|A list of related objects to populate. By default only the id of the related object will be included|

## Get without ID

|Parameter|Type|Description|
|---------|----|-----------|
|Any Column Name|String|Given a column name as the key, and a value, the resulting list will be AND filtered to match. Column values preceded with a `!` will be negatively matched|
|columns|Array|A list of columns to include on resource. By default, all visible columns will be included|
|populate|Array|A list of related objects to populate. By default only the id of the related object will be included|
|limit|Number|Number of resources to include in response. Useful for pagination. This would be the number of items per page.|
|offset|Number|Offset of the first resource to include in list. Useful for pagination. This would indicate which page to start on.|
|sort|String|Column(w) to sort on. Columns prefixed by a `-`, this will perform a Descending sort.|

## Examples

### Basic Usage

```js
// Express.js route file: routes/widgets.js
const express = require('express');
const router = express.Router();
const mongoose = require('mongoose');
const Widget = mongoose.Model('Widget');
const Rest = require('mongoose-rest-api');

const endpoints = Rest(Widget, 'widget_id'); // 'widget_id' is used to identify the ID parameter in our route definitions

router.get('/', function(req, res, next) {
    methods.get(req, res).then(result => {
        return res.status(200).json(result);
    }, err => {
        return next(err);
    });
});

router.get('/:widget_id', function(req, res, next) { // note that :widget_id matches the second argument of Rest()
    endpoints.get(req, res).then(result => {
        if(result) {
            return res.status(200).json(result);
        } else {
            return res.sendStatus(404);
        }
    }, err => {
        return next(err);
    });
});

router.post('/', function(req, res, next) {
    endpoints.post(req, res).then(result => {
        return res.status(201).json(result);
    }, err => {
        return next(err);
    });
});

router.patch('/:widget_id', function(req, res, next) {
    endpoints.patch(req, res).then(result => {
        if(result) {
            return res.status(200).json(result);
        } else {
            return res.sendStatus(404);
        }
    }, err => {
        return next(err);
    });
});

router.delete('/:widget_id', function(req, res, next) {
    endpoints.patch(req, res).then(result => {
        return res.status(200).json(result);
    }, err => {
        return res.sendStatus(200);
    });
});
```

### Parameter Validation

```js
// Express.js route file: routes/widgets.js
const express = require('express');
const router = express.Router();
const mongoose = require('mongoose');
const Widget = mongoose.Model('Widget');
const Q = require('q');
const Rest = require('mongoose-rest-api');

const endpoints = Rest(Widget, 'widget_id');

const myPost = function(req, res) {
    let deferred = Q.defer();
    validate_widget(req.body).then(result => { // validate_widget() defined elsewhere
        endpoints.post(req, res).then(result => deferred.resolve(result), err => deferred.reject(err));
    }, err => {
        deferred.reject(err);
    });
    return deferred.promise;
}

router.post('/', function(req, res, next) {
    myPost(req, res).then(result => { // Note we use myPost here instead of endpoints.post
        return res.status(200).json(result);
    }, err => {
        return next(err);
    });
});
```
