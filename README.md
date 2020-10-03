Wrapper library for the Filevine API
====================================
[![PyPI version](https://badge.fury.io/py/filevine.svg)](https://pypi.org/project/filevine)
[![Maintainability](https://api.codeclimate.com/v1/badges/bf254c3b8df16d56357b/maintainability)](https://codeclimate.com/github/W1ndst0rm/Filevine/maintainability)

[API Documentation](https://developer.filevine.io/v2/overview)

Key Features
============
* Leverages asyncio to speed up IO bound requests
* Automatically refreshes session authentication tokens
* Built-in token bucket rate limiting
* API endpoints with built-in pagination

Table of contents
=================

<!--ts-->
* [Installation](#library-instalation)
* [Getting Started](#getting-started)
    * [Using built-in endpoints](#using-built-in-endpoints)
    * [Using raw HTTP methods](#using-raw-http-methods)
    * [Base URL](#base-url)
* [Rate Limiting and Connection Management](#Rate-Limiting-and-Connection-Management)
* [Exceptions](#exceptions)
    * [FilevineHTTPException](#FilevineHTTPException)
    * [FilevineRateLimitException](#FilevineRateLimitException)
    * [FilevineTypeError](#FilevineTypeError)
    * [FilevineValueError](#FilevineValueError)
* [Examples](#examples)
<!--te-->
## Library Installation
```shell script
pip install filevine
```

Getting Started
=================
The only required parameter is a path to a yaml credentials file with the following keys: `key`, `secret`, `queueid`.
It should look like this:
```yaml
key: "fvpk_************************"
secret: "fvsk_***********************************"
queueid: "***********"
``` 
These values are obtained from the [developer portal](https://portal.filevine.io/).
The filevine module uses these credentials to obtain the accessToken and refreshToken for the authorization header.
These tokens are refreshed as needed throughout script execution. 

 
Using built-in endpoints
------------------------
```python
from filevine import Filevine
from filevine.endpoints import get_contact_list

async with Filevine(credentials_file="creds.yml") as fv:
    async for contact in get_contact_list(fv.conn, fields=['fullName', 'personId'], first_name='James'):
        print(contact['fullName'])
```
This will request the `personId` and `fullName` fields for all contacts with the first name of 'James'.

Using raw HTTP methods
----------------------
If there isn't a function written for the built-in endpoint you need, you can still use the rate limiting
and credential rotation features
```python
from filevine import Filevine

async with Filevine(credentials_file="creds.yml") as fv:
    query_parameters = {'requestedFields': ['fullName, personId'], 'firstName': 'James', 'offset': 0, 'limit': 50}
    contacts = await fv.conn.get(endpoint='/core/contacts', params=query_parameters)
    for contact in contacts:
        print(contact['fullName'])
```
This will request the `personId` and `fullName` fields for the first 50 contacts with the first name of 'James'.

POST and DELETE work similarly
```python
from filevine import Filevine

async with Filevine(credentials_file="creds.yml") as fv:
    # POST Example
    body = {
        'firstName' : 'John',
        'lastName' : 'Doe',
        'fullName' : 'John Doe',
        'gender': 'M',
        'personTypes': ['Client'] 
    }
    response = await fv.conn.post(endpoint='/core/contacts', json=body)
    # DELETE Example
    await fv.conn.delete(endpoint='/core/documents/1234')
```

Base URL
--------
The base url for the server defaults to United States server at https://api.filevine.io.
To access the Canada specific server pass in the base_url parameter
```python
from filevine import Filevine, BaseURL

# Use the built-in Enum
async with Filevine(credentials_file="creds.yml", base_url=BaseURL.CANADA) as fv:

# Pass in a string
async with Filevine(credentials_file="creds.yml", base_url='https://api.filevine.ca') as fv:
```

Rate Limiting and Connection Management
======================================= 
The built-in rate limiter uses a token bucket technique. Each  web request consumes a token,
and tokens regenerate at a set rate. The bucket has a fixed capacity to keep the initial burst of requests
from exceeding the rate-limit.

To use the built-in rate limiter, two parameters must be passed to the filevine object:
* `rate_limit_max_tokens` sets the capacity of the token bucket
* `rate_limit_token_regen_rate` sets how many tokens are regenerated per second.

If either one of the parameters is not set, no rate-limiting will occur.
```python
async with Filevine(credentials_file="creds.yml", rate_limit_max_tokens=10, rate_limit_token_regen_rate=10) as fv:
    fv.do_something()
```
Additionally, the rate limiter will use an exponential backoff algorithm to
temporarily slow down requests when the server returns a HTTP 429 error (Rate Limit Exceeded). 

Alternatively the total number of simultaneous connections to the server can limited by passing
the `max_connections` parameter. If `max_connections` is not set, the default value of `100` will be used.

Exceptions
==========
The filevine module includes several exceptions to make error handling easier.
All exceptions inherit from `FilevineException`.

FilevineHTTPException
---------------------
* Inherits from `FilevineException`
* This exception is raised when the API returns any non 200 status code. 
* Parameters:
    * code - The HTTP error code
    * url - The url accessed
    * msg - The body of the server response or `"Received non-2xx HTTP Status Code {code}"`

FilevineRateLimitException
--------------------------
* Inherits from `FilevineHTTPException`
* This exception is raised when the API returns a 429 status code (Rate Limit Exceeded). 
* Parameters:
    * code - The HTTP error code. *It will always be 429*
    * url - The url accessed
    * msg - The body of the server response or `"Received non-2xx HTTP Status Code 429"`

FilevineTypeError
-----------------
* Inherits from `FilevineException` and `TypeError`
* Raised when a parameter for an endpoint does not match the required type
* Parameters
    * msg

FilevineValueError
------------------
* Inherits from `FilevineException` and `ValueError`
* Raised when a parameter for an endpoint does not meet the requirements in the endpoint specification.
For example, a contact's personTypes must be in the list `['Adjuster', 'Attorney', 'Client', 'Court',
'Defendant', 'Plaintiff', 'Expert', 'Firm', 'Insurance Company', 'Involved Party', 'Judge', 'Medical Provider']`
* Parameters:
    * msg
    
Examples
========
The [examples](examples/README.md) folder contains complete non-trivial examples of this module in use.
