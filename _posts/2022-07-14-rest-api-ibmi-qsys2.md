---
date: 2022-07-14 15:30:55 -05:00
title: Utilize a RESTful API with IBMi QSYS_HTTP Tools in SQL
categories: [IBM i,]
tags: [ibmi, api, qsys2, rest, sql]
---

## The Api

This demo used the `fakeStoreApi` which is a free online REST API that you can use whenever you need Pseudo-real data for without running any server-side code. It's awesome for teaching purposes, sample codes, tests, etc. This API does not require authentication for requests

> A list of other public APIs can be found [here](https://github.com/public-apis/public-apis)

## HTTP Functions Overview
These HTTP functions are used to make HTTP requests that use web services. These functions allow the SQL programmer to use Representational State Transfer (RESTful) via SQL, including Embedded SQL. They provide the same capabilities as the [SYSTOOLS HTTP functions](https://www.ibm.com/docs/en/ssw_ibm_i_75/rzajq/rzajqhttpoverview.htm) without the overhead of creating a JVM.

These HTTP functions exist in QSYS2 and have lower overhead than the SYSTOOLS HTTP functions. Additional benefits of the QSYS2 HTTP functions are HTTP authentication, proxy support, configurable redirection attempts, and configurable SSL options.

The URL parameter supports http: and https: URLs. The https: URL indicates that network communication should take place over a secure communication channel. An https request uses TLS (Transport Layer Security) to create the secure channel. This secure channel encrypts any transmitted data and also prevents man-in-the-middle attacks. Any communication that contains secure information should use https instead of http. Because of the sensitive nature of userids and passwords, HTTP authentication is not allowed for http URLs.

### Foundational HTTP functions
The foundational functions are named according to the two dimensions used when making HTTP requests.
The first dimension is the HTTP operation. There are 5 different HTTP operations: GET, PUT, POST, PATCH, and DELETE.
The second dimension indicates whether the verbose version of the function should be used. The non-verbose functions are scalar functions that return the response as a CLOB. The verbose functions are table functions that return a single row, which includes the return header information that is sent from the HTTP server. The header information is formatted as JSON.
The names of the functions reflect these dimensions. For example, HTTP_GET_VERBOSE uses the GET operation from the first dimension and the VERBOSE setting from the second dimension. All the functions return CLOB data.

> See the [IBM Docs](https://www.ibm.com/docs/en/i/7.4?topic=programming-http-functions-overview) for more details

# Get a List of Products

The first demo receives a list of products from the `fakeStoreApi` as JSON. Here is a look at the JSON we can expect to receive:

```json
[
  {
    "id": 1,
    "title": "Fjallraven - Foldsack No. 1 Backpack, Fits 15 Laptops",
    "price": 109.95,
    "description": "Your perfect pack for everyday use and walks in the forest. Stash your laptop (up to 15 inches) in the padded sleeve, your everyday",
    "category": "men's clothing",
    "image": "https://fakestoreapi.com/img/81fPKd-2AYL._AC_SL1500_.jpg",
    "rating": {
      "rate": 3.9,
      "count": 120
    }
  },
  {
    "id": 2,
    "title": "Mens Casual Premium Slim Fit T-Shirts ",
    "price": 22.3,
    "description": "Slim-fitting style, contrast raglan long sleeve, three-button henley placket, light weight & soft fabric for breathable and comfortable wearing. And Solid stitched shirts with round neck made for durability and a great fit for casual fashion wear and diehard baseball fans. The Henley style round neckline includes a three-button placket.",
    "category": "men's clothing",
    "image": "https://fakestoreapi.com/img/71-3HjGNDUL._AC_SY879._SX._UX._SY._UY_.jpg",
    "rating": {
      "rate": 4.1,
      "count": 259
    }
  }
]
```
This is actually an array of two JSON objects. The JSON tools provided by DB2 are smart and will know to treat each object separately. To make sure we are getting the expected JSON we can print the results of `QSYS2.HTTP_GET` with the `VALUES` keyword.

```sql
VALUES QSYS2.HTTP_GET(
            'http://fakestoreapi.com/products?limit=2',
           ''
           );
```

`QSYS2.HTTP_GET` takes two arguments. The first argument is the URL of the API endpoint the GET request will be sent to. In this case it is the `http://fakestoreapi.com/products` endpoint and the `limit=2` parameter is added to only get two items total. The second argument is for [HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers) parameters. In this case no HTTP header parameters need to be specified to complete the request so this field can be left empty.

The `JSON_TABLE` function can take any `QSYS2.HTTP_XXXX` that produces JSON as an argument. The values from the JSON keys can then be extracted and placed directly into a table. 

```sql
SELECT *
FROM JSON_TABLE(
        QSYS2.HTTP_GET(
            'http://fakestoreapi.com/products?limit=10',
            ''
        ),
        '$' COLUMNS(
            name VARCHAR(75) PATH 'lax $.title',
            totalRatings INT PATH 'lax $.rating.count'
        )
    );
```
This example will extract the values from the `title` (name of item) key and the nested field `count` (total number of ratings) key in the ratings array. and then place them into a table with the column names `name` and `totalRatings`.

#### Result:

| Name                                                                        | Rating |
| --------------------------------------------------------------------------- | ------ |
| Fjallraven - Foldsack No. 1 Backpack, Fits 15 Laptops                       | 120    |
| Mens Casual Premium Slim Fit T-Shirts                                       | 259    |
| Mens Cotton Jacket                                                          | 500    |
| Mens Casual Slim Fit                                                        | 430    |
| John Hardy Women's Legends Naga Gold & Silver Dragon Station Chain Bracelet | 400    |
| Solid Gold Petite Micropave                                                 | 70     |
| White Gold Plated Princess                                                  | 400    |
| Pierced Owl Rose Gold Plated Stainless Steel Double                         | 100    |
| WD 2TB Elements Portable External Hard Drive - USB 3.0                      | 203    |
| SanDisk SSD PLUS 1TB Internal SSD - SATA III 6 Gb/s                         | 470    |

# POST a New User

The next example is sending a POST request to the `https://fakestoreapi.com/users` endpoint to create a new user. The `fakeStoreApi` [docs](https://fakestoreapi.com/docs) indicate that the body of our HTTP POST request should contain the following JSON object:

```json
{
    "email":"John@gmail.com",
    "username":"johnd",
    "password":"m38rmF$",
    "name":{
        "firstname":"John",
        "lastname":"Doe"
    },
    "address":{
        "city":"kilcoole",
        "street":"7835 new road",
        "number":3,
        "zipcode":"12926-3874",
        "geolocation":{
            "lat":"-37.3159",
            "long":"81.1496"
        }
    },
    "phone": "1-570-236-7033"
}
```
Upon a successful add of a new user the API will return a 200 response code and a JSON object with the user's new id:

```json
{
  "address": {
    "geolocation": {
      "lat": "-37.3159",
      "long": "81.1496"
    },
    "city": "kilcoole",
    "street": "7835 new road"
  },
  "_id": "62c73539f0321700139f4682",
  "id": 1,
  "email": "John@gmail.com",
  "username": "johnd",
  "password": "m38rmF$",
  "phone": "1-570-236-7033"
}
```

`QSYS2.HTTP_POST` takes three This time arguments the URL, the HTTP body and the HTTP header parameters. This time the URL and body arguments of the `QSYS2.HTTP_POST` will be assigned to variables for readability. 

We also need to specify the "Content-Type" in the HTTP header to indicate that our HTTP body will be in JSON format. By default `QSYS2.HTTP_POST` specifies the content type of the body to be XML. To override this setting we pass in the header settings in JSON format:

```json
{"header":"Content-Type,application/json;charset=utf-8"}
```
> More information about the different header settings that can to passed to the `QSYS2.HTTP_XXXX` tools can be found in the [IBM docs](https://www.ibm.com/docs/en/i/7.4?topic=functions-http-get#rbafzscahttpget__HTTP_options)

Here all the moving parts put together:

```sql
Create or replace variable @userURL varchar(50) ;
SET @userURL = 'http://fakestoreapi.com/users';

Create or replace variable @postBody varchar(500) ;
SET @postBody = '{
    "email":"John@gmail.com",
    "username":"johnd",
    "password":"m38rmF$",
    "name":{
        "firstname":"John",
        "lastname":"Doe"
    },
    "address":{
        "city":"kilcoole",
        "street":"7835 new road",
        "number":3,
        "zipcode":"12926-3874",
        "geolocation":{
            "lat":"-37.3159",
            "long":"81.1496"
        }
    },
    "phone": "1-570-236-7033"
}';

SELECT *
FROM JSON_TABLE(
    QSYS2.HTTP_POST(
        @userURL,
        @postBody,
        '{"header":"Content-Type,application/json;charset=utf-8"}'
    ),
    '$' COLUMNS(
        testID INT PATH 'lax $.id',
        email VARCHAR(50) PATH 'lax $.email',
        username VARCHAR(50) PATH 'lax $.username',
        password VARCHAR(50) PATH 'lax $.password'
    )
);
```
And the result:

| TESTID | EMAIL          | USERNAME | PASSWORD |
| ------ | -------------- | -------- | -------- |
| 1      | John@gmail.com | johnd    | m38rmF$  |