# odata-client

A client library for accessing odata resources using node.  HTTP queries return a promise.

## Installation

`npm install odata-client`

## Usage

```
const odata = require('odata-client');
var q = odata({service: 'https://example.com', resources: 'Customers'});
q.top(5).skip(10).filter('Balance gt 5000').and('CreditLimit', '<', 10000).get()
.then(function(response) {
  ...
});
```

## odata object

* `odata(config)`

The `odata(config)` function produces a query object for the construction of queries. `config` is an object 
with the following options:

  1. `service` - the base URL of the service

  1. `resources` - the resource part of the URL for the query, e.g. `Customers` or `Customers('ACME01')/Orders`.
You can also add resource parts using the `resource` method of the query function

  1. `custom` - optional object containing addition query parameters, e.g. `{access_token:'123456'}` will append 
`?access_token=123456` to the query URL.

  1. `version` - set OData-Version HTTP header

  1. `maxVersion` - set OData-MaxVersion HTTP header

  1. `format` - specify response format (e.g. `json`)

* `expression(left, op, right)`

Used to produce subexpressions in a filter.  For example, `q.filter('CreditBalance', '>', odata.expression('OrderValue', '+', 100))`
produces `$filter=CreditBalance gt (OrderValue add 100)`. The arguments are the same as for `q.filter`, see below.

Expressions can be chained, eg

```
expression('Balance', '>', 500').and('CreditLimit', '=', 0) // (Balance gt 500) and (CreditLimit eq 0)
expression('Balance', '+', 1000).lt('CreditLimit') // (Balance add 1000) lt (CreditLimit)
```

* `identifier(string)`
* `literal(string)`

In a filter expression part, the left argument is normally treated as an identifier (i.e., if it's a string it isn't
surrounded by quotes) whereas the right argument is assumed to be a literal (strings are surrounded by quotes).  These 
methods allow you to override this. e.g.

```
q.filter(odata.literal('Customer'), '=', odata.identifer('Type')) // $filter='Customer' eq Type
```

## query object

The query object has the following methods:

* `top(n)`

Adds a `$top=n` query parameter.

* `skip(n)`

Adds a `$skip=n` query parameter.

* `filter(left, op, right)`

Used for constructing `$filter` requests. There are several ways to call this method:

  1. If called with a string as the sole argument, the string is used as a literal filter, e.g.
`q.filter("Account eq 'ACME01'")`

  1. If all three arguments are specified, `op` should be one of the usual odata operations such as `eq` or `add`,
or the symbolic equivalents such as `=` or `+`. e.g. `q.filter('Account', '=', 'ACME01')`. The `left` and `right`
arguments can be `odata.expression`s for building nested queries. 

  1. If called with two arguments, the operator is assumed to be `eq`, e.g. `q.filter('Account', 'ACME01')`

The `left` argument is assumed to be an identifier while `right` is assumed to be a literal, which affects the
quoting of strings.  You can override this behaviour with the `odata.literal` and `odata.identifier` functions, see above.

If two or more filters are chained, they are `and`ed together. `q.filter('Balance gt 1000').filter('Status', 'stop')`
profduces `$filter=(Balance gt 1000) and (Status eq 'stop')`.

* `and(left, op, right)`

Synonym for `filter`.

* `or(left, op, right)`

Adds an `or` clause to the filter being built.

* `not(left, op, right)`

Adds a `not` clause to the filter

* `all(field, property, op, value)`

Adds an `all` filter, e.g. 

```
q.all('Orders', 'Value', '<', 50) // ?$filter=Orders/all(p0:p0/Value lt 50)
```

* `any(field, property, op, value)`

Adds an `any` filter, e.g. 

```
q.any('Orders', 'Lines/$count', '>=', 10) // ?$filter=Orders/any(p0:p0/Lines/$count ge 10)
```

* `resource(resource, value)`

Adds a new part to the resource section. e.g.

```
odata({service: 'https://example.com'}).resource('Customers').resource('Orders'); // https://example.com/Customers/Orders
odata({service: 'https://example.com'}).resource('Customers', 'ACME01').resource('Orders'); // https://example.com/Customers('ACME01')/Orders
odata({service: 'https://example.com'}).resource('Customers', {account:'ACME01'}).resource('Orders'); // https://example.com/Customers(account='ACME01')/Orders
```

* `select(items)`

Adds a `$select` clause to the filter, e.g. `q.select('Account', 'Status')` produces `$select=Account,Status`.

* `expand(item)`

Adds an item to `$expand`

* `search(term)`

Sets the term for `$search`.

* `count`

Adds a $count clause to the query

* `orderby(item, dir)`

Adds an `$orderby` clause to the query.  There are several ways to call this function:

  1. `q.orderby('Account')` produces `$orderby=Account`

  1. `q.orderby('Account', 'desc')` produces `$orderby=Account desc`

  1. `q.orderby(['Status', 'desc'], ['Account'])` produces `$orderby=Status desc,Account`

* `custom(name, value)`

Adds custom query prameters to the query using either a pair of parameters or an object, e.q.

```
q.custom('access_token', '123456') // ?access_token=123456
q.custom({access_token: '123456', version: '1.2'}) // ?access_token=123456&version=1.2
```

* `query`

Produces the query string, e.g. 

```
odata({service: 'https://example.com/Customers'}).top(5).query() // 'https://example.com/Customers?$top=5'
```

* `get`, `post`, `put`, `patch`, `delete`

Perform an HTTP operation, returning a promise which resolves to an HTTP response.  Each function can take an `options`
argument which is passed to the undelying [request](https://www.npmjs.com/package/request) library.

