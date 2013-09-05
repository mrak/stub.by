# Stubby

```
a small web server for stubbing external systems during development
```

##### Why the word "stubby"?
It is a stub HTTP server after all, hence the "stubby". Also, in Australian slang "stubby" means _beer bottle_

## Table of Contents

__stubby4j__: [![Build Status](https://secure.travis-ci.org/azagniotov/stubby4j.png?branch=master)](http://travis-ci.org/azagniotov/stubby4j)

__stubby4node__: [![Build Status](https://secure.travis-ci.org/mrak/stubby4node.png?branch=master)](http://travis-ci.org/Afmrak/stubby4node)
[![NPM version](https://badge.fury.io/js/stubby.png)](http://badge.fury.io/js/stubby)

* [Key Features](#key-features)
* [Why would a developer use stubby](#why-would-a-developer-use-stubby)
* [Why would a QA use stubby](#why-would-a-qa-use-stubby)
* [Adding stubby to your project](#adding-stubby-to-your-project)
* [Command-line Switches](#command-line-switches)
* [Endpoint Configuration HOWTO](#endpoint-configuration)
   * [Request](#request)
   * [Response](#response)
   * [Dynamic token replacement in stubbed response](#dynamic-token-replacement-in-stubbed-response)
* [The Admin Portal](#the-admin-portal)
* [The Stubs Portal](#the-stubs-portal)
* [Programmatic API](#programmatic-api)

## Key Features
* Emulates external webservices in a sandbox for your application to consume over HTTP(S)
* HTTP request verification and HTTP response stubbing
* Regex support for dynamic matching on URI, query params, headers, POST body (ie:. `mod_rewrite` in Apache)
* Dynamic flows: Multiple stubbed responses on the same stubbed URI to test multiple application flows
* Fault injection, where after X good responses on the same URI you get a bad one
* Serve binary files as stubbed response content (images, PDFs. etc.)
* Embed stubby to create a web service sandbox for your integration test suite

### Incubating features (experimental or not present in all versions)
* Dynamic token replacement in stubbed responses by leveraging regex capturing groups as token values during HTTP request verification
* Record & Replay. The HTTP response is recorded on the first call, having the subsequent calls play back the recorded HTTP response, without actually connecting to the external server

## Why would a developer use stubby?
#### You want to:
* Simulate responses from real server and don't care to (or cannot) go over the network
* Stub third party web services that your application contacts which are not yet ready
* Verify that your code makes HTTP requests with all the required parameters and/or headers
* Verify that your code correctly handles HTTP error codes
* Trigger a response from the server based on the request parameters over HTTP or HTTPS
* Support any of the available HTTP methods
* Simulate support for Basic Authentication
* Support HTTP 30x redirects
* Provide canned answers in your contract/integration tests
* Enable delayed responses for performance and stability testing
* Avoid to spend time coding for the above requirements
* Concentrate on the task at hand

## Why would a QA use stubby?
#### You want to:
* Specify mock responses to simulate page conditions without real data
* Test polling mechanisms by stubbing a sequence of responses for the same URI
* Easily swap data config files to run different data sets and responses
* Have an all-in-one stub server to handle mock data with less need to upkeep code for test generation

## Command-line Switches
```
Standard switches:
 -a,--admin <arg>      Port for admin portal. Defaults to 8889.
 -d,--data <arg>       Data file to pre-load endoints. YAML or JSON format.
 -h,--help             This help text.
 -l,--location <arg>   Hostname at which to bind stubby.
 -m,--mute             Prevent stubby from printing to the console.
 -s,--stubs <arg>      Port for stub portal. Defaults to 8882.
 -t,--tls <arg>        Port for https stubs portal. Defaults to 7443.
 -w,--watch            Auto-reload data file when edits are made.
 -v,--version          Prints stubby's version number.

stubby4j switches:
 -p,--password <arg>   Password for the provided keystore file.
 -k,--keystore <arg>   Keystore file for custom SSL. By default SSL is
                       enabled using internal keystore.

stubby4node switches:
 -p,--pfx <arg>        PFX file. Ignored if used with --key/--cert
 -k,--key <arg>        Private key file. Use with --cert.
 -c,--cert <arg>       Certificate file. Use with --key.
```

## Endpoint Configuration

This section explains the usage, intent and behavior of each property on the `request` and `response` objects.

Here is a fully-populated, unrealistic endpoint:
```yaml
- request:
    url: ^/your/awesome/endpoint$
    method: POST
    query:
      exclamation: post requests can have query strings!
    headers:
      content-type: application/xml
    post: >
      <!xml blah="blah blah blah">
      <envelope>
        <unaryTag/>
      </envelope>
    file: tryMyFirst.xml
  response:
  - status: 200
    latency: 5000
    headers:
      content-type: application/xml
      server: stubbedServer/4.2
    body: >
      <!xml blah="blah blah blah">
      <responseXML>
        <content></content>
      </responseXML>
    file: responseData.xml
  - status: 200
    body: "Haha!"
```

### request

This object is used to match an incoming request to stubby against the available endpoints that have been configured.

#### url (required)

* is a full-fledged __regular expression__
* This is the only required property of an endpoint.
* signify the url after the base host and port (i.e. after `localhost:8882`).
* any query paramters are stripped (so don't include them, that's what `query` is for).
    * `/url?some=value&another=value` becomes `/url`
* no checking is done for URI-encoding compliance.
    * If it's invalid, it won't ever trigger a match.

This is the simplest you can get:
```yaml
- request:
    url: /
```

A demonstration using regular expressions:
```yaml
- request:
    url: ^/has/to/begin/with/this/

- request:
    url: /has/to/end/with/this/$

- request:
    url: ^/must/be/this/exactly/with/optional/trailing/slash/?$

- request:
    url: ^/[a-z]{3}-[a-z]{3}/[0-9]{2}/[A-Z]{2}/[a-z0-9]+$
```

#### method

* defaults to `GET`.
* case-insensitive.
* can be any of the following:
    * HEAD
    * GET
    * POST
    * PUT
    * POST
    * DELETE
    * etc.

```yaml
- request:
    url: /anything
    method: GET
```

* it can also be an array of values.

```yaml
- request:
    url: /anything
    method: [GET, HEAD]

- request:
    url: /anything
    method:
      - GET
      - HEAD
      - POST
```

#### query

* values are full-fledged __regular expressions__
* if ommitted, stubby ignores query parameters for the given url.
* a yaml hashmap of variable/value pairs.
* allows the query parameters to appear in any order in a uri

```yaml
-  request:
      method: GET
      url: ^/with/parameters$
      query:
         type_name: user
         client_id: id
         client_secret: secret
         random_id: "^sequence/-/\\d/"
         session_id: "^user_\\d{32}_local"
```

* The following will match either of these:
    * `/with/parameters?search=search+terms&filter=month`
    * `/with/parameters?filter=month&search=search+terms`

```yaml
-  request:
      url: ^/with/parameters$
      query:
         search: search terms
         filter: month
```

#### post

* is a full-fledged __regular expression__
* if ommitted, any post data is ignored.
* represents the body POST of incoming request, ie.: form data

```yaml
- request:
    url: ^/post/form/data$
    post: name=John&email=john@example.com
```

```yaml
- request:
    method: [POST]
    url: /uri/with/post/regex
    post: "^[\\.,'a-zA-Z\\s+]*$"
```

```yaml
-  request:
      url: ^/post/form/data$
      post: "^this/is/\\d/post/body"
```

#### file

* holds a path to a local file (absolute or relative to the YAML specified in `-d` or `--data`)
* if supplied, replaces `post` with the contents from the provided file
    * paths are relative from where the `--data` file is located
* if the local file could not be loaded for whatever reason (ie.: not found), stubby falls back to `post` for matching.
* allows you to split up stubby data across multiple files instead of making one huge bloated main YAML

```yaml
-  request:
      url: ^/match/against/file$
      file: postedData.json
      post: '{"fallback":"data"}'
```

postedData.json
```json
{"fileContents":"match against this if the file is here"}
```

* if `postedData.json` doesn't exist on the filesystem when `/match/against/file` is matched in incoming request, stubby will match post contents against `{"fallback":"data"}` (from `post`) instead.

#### headers

* values are full-fledged __regular expressions__
* a hashmap of header/value pairs similar to `query`.
* if ommitted, stubby ignores headers for the given url
* if stubbed, stubby will try to match __only__ the supplied headers and will ignore other headers of incoming request. In other words, the incoming request __must__ contain stubbed header values
* headers are case-insensitive during matching

The following endpoint only accepts requests with `application/json` post values:

```yaml
-  request:
      url: /post/json
      method: post
      headers:
         content-type: application/json
         x-custom-header: "^this/is/\d/test"
         x-custom-header-2: "^[a-z]{4}_\\d{32}_(local|remote)"
```

### response

Assuming a match has been made against the given `request` object, data from `response` is used to build the stubbed response back to the client.

* Can be a single response or a sequence of responses
* When sequenced responses are configured, upon each incoming request to the same URI a subsequent response in the list will be sent to the client. The sequenced responses play in a cycle 

```yaml
- request:
    method: [GET,POST]
    url: /invoice/123

  response:
    status: 201
    headers:
      content-type: application/json
    body: OK


- request:
    method: [GET]
    url: /uri/with/sequenced/responses

  response:
  - status: 201
    headers:
      content-type: application/json
    body: OK

  - status: 201
      headers:
        content-stype: application/json
      body: Still going strong!

  - status: 500
    headers:
      content-type: application/json
    body: OMG!!!


- request:
    method: [GET]
    url: /uri/with/sequenced/responses/infile

  response:
  - status: 201
    headers:
      content-type: application/json
    file: ../json/sequenced.response.ok.json

  - status: 201
    headers:
      content-stype: application/json
    file: ../json/sequenced.response.goingstrong.json

  - status: 500
    headers:
      content-type: application/json
    file: ../json/sequenced.response.omfg.json

- request:
    method: [GET]
    url: /uri/with/single/sequenced/response

  response:
  - status: 201
    headers:
      content-stype: application/json
    body: Still going strong!
```

#### status

* the HTTP status code of the response.
* integer or integer-like string.
* defaults to `200`.

```yaml
- request:
    url: ^/im/a/teapot$
    method: POST
  response:
    status: 420
```

#### body

* contents of the response body
* defaults to an empty content body

```yaml
- request:
    url: ^/give/me/a/smile$
  response:
    body: ':)'
```

```yaml
- request:
    url: ^/give/me/a/smile$

  response:
    status: 200
    body: >
      {"status": "hello world with single quote"}
    headers:
      content-type: application/json
```

```yaml
- request:
    method: GET
    url: /atomfeed/1

  response:
    headers:
      content-type: application/xml
    status: 200
    body: <?xml version="1.0" encoding="UTF-8"?><payment><paymentDetail><invoiceTypeLookupCode/></paymentDetail></payment>
```

```yaml
- request:
    url: /1.1/direct_messages.json
    query:
      since_id: 240136858829479935
      count: 1
  response:
    headers:
      content-type: application/json
    body: https://api.twitter.com/1.1/direct_messages.json?since_id=240136858829479935&count=1
```

#### file

* similar to `request.file`, but the contents of the file are used as the response `body`
* if the file could not be loaded, stubby falls back to the value stubbed in `body`
* if `body` was not stubbed, an empty string is returned by default
* it can be an ASCII or binary file (PDF, images, etc.)

```yaml
- request:
    url: /
  response:
    file: extremelyLongJsonFile.json
```

#### headers

* similar to `request.headers` except that these are sent back to the client.
* by default, the header `x-stubby-resource-id` containing the admin portal resource ID is returned with each stubbed response. The ID is useful if the returned resource needs to be updated at run time by ID via the admin portal

```yaml
- request:
    url: ^/give/me/some/json$
  response:
    headers:
      content-type: application/json
    body: >
         [{
            "name":"John",
            "email":"john@example.com"
         },{
            "name":"Jane",
            "email":"jane@example.com"
         }]
```

#### latency

* time to wait, in milliseconds, before sending back the response
* good for testing timeouts, or slow connections

```yaml
- request:
    url: ^/hello/to/jupiter$
  response:
    latency: 800000
    body: Hello, World!
```

## The Admin Portal

The admin portal is a RESTful(ish) endpoint running on `localhost:8889`. Or wherever you described through stubby's command line args.

### Supplying Endpoints to Stubby

Submit `POST` requests to `localhost:8889` or load a data-file (using -d / --data flags) with the following structure for each endpoint:

* `request`: describes the client's call to the server
   * `method`: GET/POST/PUT/DELETE/etc.
   * `url`: the URI regex string.
   * `query`: a key/value map of query string parameters included with the request. Query param value can be regex.
   * `headers`: a key/value map of headers the server should respond to. Header value can be regex.
   * `post`: a string matching the textual body of the response. Post value can be regex.
   * `file`: if specified, returns the contents of the given file as the request post. If the file cannot be found at request time, **post** is used instead
* `response`: describes the server's response (or array of responses, refer to the examples) to the client
   * `headers`: a key/value map of headers the server should use in it's response.
   * `latency`: the time in milliseconds the server should wait before responding. Useful for testing timeouts and latency
   * `file`: if specified, returns the contents of the given file as the response body. If the file cannot be found at request time, **body** is used instead
   * `body`: the textual body of the server's response to the client
   * `status`: the numerical HTTP status code (200 for OK, 404 for NOT FOUND, etc.)


#### YAML (file only or POST/PUT)
```yaml
- request:
    url: ^/path/to/something$
    method: POST
    headers:
      authorization: "bob:password"
      x-custom-header: "^this/is/\d/test"
    post: this is some post data in textual format
  response:
    headers:
      Content-Type: application/json
    latency: 1000
    status: 200
    body: You're request was successfully processed!

- request:
    url: ^/path/to/anotherThing
    query:
      a: anything
      b: more
      custom: "^this/is/\d/test"
    method: GET
    headers:
      Content-Type: application/json
    post:
  response:
    headers:
      Content-Type: application/json
      Access-Control-Allow-Origin: "*"
    status: 204
    file: path/to/page.html

- request:
    url: ^/path/to/thing$
    method: POST
    headers:
      Content-Type: application/json
    post: this is some post data in textual format
  response:
    headers:
      Content-Type: application/json
    status: 304
```


#### JSON (file or POST/PUT)
```json
[
  {
    "request": {
      "url": "^/path/to/something$",
      "post": "this is some post data in textual format",
      "headers": {
         "authorization": "bob:password"
      },
      "method": "POST"
    },
    "response": {
      "status": 200,
      "headers": {
        "Content-Type": "application/json"
      },
      "latency": 1000,
      "body": "You're request was successfully processed!"
    }
  },
  {
    "request": {
      "url": "^/path/to/anotherThing",
      "query": {
         "a": "anything",
         "b": "more"
      },
      "headers": {
        "Content-Type": "application/json"
      },
      "method": "GET"
    },
    "response": {
      "status": 204,
      "headers": {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*"
      },
      "file": "path/to/page.html"
    }
  },
  {
    "request": {
      "url": "^/path/to/thing$",
      "headers": {
        "Content-Type": "application/json"
      },
      "post": "this is some post data in textual format",
      "method": "POST"
    },
    "response": {
      "status": 304,
      "headers": {
        "Content-Type": "application/json"
      }
    }
  }
]
```

If you want to load more than one endpoint via file, use either a JSON array or YAML list (-) syntax. When creating or updating one stubbed request, the response will contain `Location` in the header with the newly created resources' location

### Getting the ID of a Loaded Endpoint

Stubby adds the response-header `x-stubby-resource-id` to outgoing responses. This ID can be referenced for use with the Admin portal.

### Getting the Current List of Stubbed Endpoints

Performing a `GET` request on `localhost:8889` will return a YAML list of all currently saved responses. It will reply with `204 : No Content` if there are none saved.

Performing a `GET` request on `localhost:8889/<id>` will return the YAML object representing the response with the supplied id.

#### The Status Page

You can also view the currently configured endpoints by going to `localhost:8889/status`

#### The Refresh Stubbed Data Endpoint

If for some reason you do not want/cannot/not able to use `--watch` flag when starting stubby4j (or cannot restart),
you can submit `GET` request to `localhost:8889/refresh` (or load it in a browser) in order to refresh the stubbed data.

### Changing Existing Endpoints

Perform `PUT` requests in the same format as using `POST`, only this time supply the id in the path. For instance, to update the response with id 4 you would `PUT` to `localhost:8889/4`.

### Deleting Endpoints

Send a `DELETE` request to `localhost:8889/<id>`

## The Stubs Portal

Requests sent to any url at `localhost:8882` (or wherever you told stubby to run) will search through the available endpoints and, if a match is found, respond with that endpoint's `response` data

### How Endpoints Are Matched

For a given endpoint, stubby only cares about matching the properties of the request that have been defined in the YAML. The exception to this rule is `method`; if it is omitted it is defaulted to `GET`.

For instance, the following will match any `POST` request to the root url:

```yaml
- request:
    url: /
    method: POST
  response: {}
```

The request could have any headers and any post body it wants. It will match the above.

Pseudocode:

```
for each <endpoint> of stored endpoints {

   for each <property> of <endpoint> {
      if <endpoint>.<property> != <incoming request>.<property>
         next endpoint
   }

   return <endpoint>
}
```


## Programmatic APIs

Since programming languages vary, please refer to the individual project pages for their respective programmatic interfaces.
