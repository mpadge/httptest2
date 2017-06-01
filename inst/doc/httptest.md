<!--
%\VignetteEngine{knitr::knitr}
%\VignetteIndexEntry{httptest: A Test Environment for HTTP Requests in R}
-->

# httptest: A Test Environment for HTTP Requests in R



Testing code and packages that communicate with remote servers can be painful. Dealing with authentication, bootstrapping server state, cleaning up objects that may get created during the test run, network flakiness, and other complications can make testing seem too costly to bother with. But it doesn't need to be that hard. The `httptest` package enables one to test all of the logic on the R sides of the API in your package without requiring access to the remote service. Importantly, `httptest` provides three test **contexts** that mock the network connection in different ways, and it offers additional **expectations** to assert that HTTP requests were--or were not--made. The package also includes a context for recording the responses of real requests and storing them as fixtures that you can later load in a test run. Using these tools, one can test that code is making the intended requests and that it handles the expected responses correctly, all without depending on a connection to a remote API.

# Using `with_mock_API`

In this context, HTTP requests attempt to load API responses from
files. If the fixture file exists, it is loaded and returned as the response; if it does not, an error with a well-defined message containing the request information is raised, and we can write tests that look for that error. These two different modes allow us to make assertions about two different kinds of logic: (1) given some inputs, does my code make the correct request(s) to that service; and (2) does my code correctly handle the types of responses that that service can return?

## Example

The `twitteR` package is quite popular ([14k downloads per month](https://cranlogs.r-pkg.org/badges/twitteR)), but like many R packages, it doesn't have a test suite. It is hard to test the querying of an API that requires OAuth authentication, and thus a user/account to use in testing. But `httptest` can fix that.

First, we need to set up some boilerplate for `testthat` testing, which `httptest` uses. Add a "tests" directory, and in that, a "testthat.R" with

    library(httptest)
    test_check("twitteR")

Note that it says `library(httptest)` instead of the `library(testthat)` that you would otherwise use with `testthat`. Next, let's add a "testthat" directory to hold our test files. In that directory, we'll start with a "helper.R" file for common setup code. From some experimenting, it's clear that the package checks to see if you have an OAuth token set. We're not actually going to hit the Twitter API, so we don't need a real token, we just need a token to exist:

    use_oauth_token("foo")

Now let's write a test. Add "test-user.R" and put a test for getting a user record, which according to the source code, should hit the "show user" Twitter API, documented [here](https://dev.twitter.com/rest/reference/get/users/show).

    context("Get a user")

    with_mock_API({
        test_that("We can get a user object", {
            user <- getUser("twitterdev")
        })
    })

When we run the tests, it fails with

    Get a user: Error: GET https://api.twitter.com/1.1/users/show.json?screen_name=twitterdev (api.twitter.com/1.1/users/show.json-84627b.json)

The error message reveals a few things about how `with_mock_API` works. First, the it tells us what the request method and URL was, and if there were a request body, it would be part of the error message as well. We can make assertions about the expected request based on that error message--more on that below. Second, the final part of the error message is a file name. That's the mock file that the test context was looking for and didn't find.

Requests are translated to mock file paths according to several rules that
incorporate the request method, URL, query parameters, and body. First, the URL is modified in two ways in order to allow it to map to a
local file system. All mock files have the request protocol such as "http://"
removed from the URL, and they also have a file extension appended. In an
HTTP API, a "directory" itself is a resource,
so the extension allows distinguishing directories and files in the file
system. That is, a mocked `GET("http://example.com/api/")` may read a
"example.com/api.json" file, while
`GET("http://example.com/api/object1/")` reads "example.com/api/object1.json".

The extension also gives information on content type. Two extensions are
currently supported: (1) .json and (2) .R. JSON mocks can be stored in .json
files, and when they are loaded by `with_mock_API`, relevant request
metadata (headers, status code, etc.) are inferred. If your API doesn't
return JSON, or if you want to simulate requests with other behavior (201
Location response, or 400 Bad Request, for example), you can store full
`response` objects in .R files that `with_mock_API` will `source` to load.
Any request can be stored as a .R mock, but the .json mocks offer a
simplified, more readable alternative.

Second, if the request URL contains a query string, it will be popped off,
hashed, and the first six characters appended to the
file being read. For example, `GET("api/object1/?a=1")` reads
"api/object1-b64371.json". Third, request bodies are similarly hashed and
appended. Finally, if a request method other than GET is used it will be
appended to the end of the end of the file name. For example,
`POST("api/object1/?a=1")` reads "api/object1-b64371-POST.json".

Back to the example. The error message tells us that the request it is making is what we should expect based on the API documentation, so that's good. The API documentation page has an example JSON response, so let's copy that to the fixture file path that the message indicated. The API response looks like

    {
      "id": 2244994945,
      "id_str": "2244994945",
      "name": "TwitterDev",
      "screen_name": "TwitterDev",
      "location": "Internet",
      "profile_location": null,
      "description": "Developer and Platform Relations @Twitter. We are developer advocates. We can't answer all your questions, but we listen to all of them!",
      "url": "https://t.co/66w26cua1O",
      ...

When we run the tests again, there's no more error. Great! Now let's assert some things about the user object we have:

    test_that("We can get a user object", {
        user <- getUser("twitterdev")
        expect_is(user, "user")
        expect_identical(user$name, "TwitterDev")
        expect_output(print(user), "TwitterDev")
    })

We can also assert that we're making the right shaped request by looking for a mock we don't have. Use `expect_GET` to look for a GET request, and include the expected request URL:

    test_that("GET user request is the right shape", {
        expect_GET(getUser("enpiar"),
            "https://api.twitter.com/1.1/users/show.json?screen_name=enpiar")
    })

Now, we can do the same for the `lookupUsers` function. It should hit the `users/lookup.json` endpoint and the function should return a list of `user` objects:

    test_that("lookupUsers retrieves many", {
        result <- lookupUsers(c("twitterapi", "twitter"))
        expect_is(result, "list")
        expect_true(all(vapply(result, inherits, logical(1), what="user")))
    })

Drop the example response from the [API documentation](https://dev.twitter.com/rest/reference/get/users/lookup) in the right location, and that passes as well.

We just went from zero tests to 25 percent line coverage in a few minutes, using about 20 lines of code. We've tested a lot of the code that prepares the requests of the user API, and we've tested much of the code that handles the server's response, the "user" objects that get created in R, and their methods. Our resulting test directory, containing both our test files and our API fixtures, looks like this:

    tests
    ├── testthat
    │   ├── api.twitter.com
    │   │   └── 1.1
    │   │       └── users
    │   │           ├── lookup.json-342984.json
    │   │           └── show.json-84627b.json
    │   ├── helper.R
    │   └── test-user.R
    └── testthat.R

# Recording mocks with `capture_requests`

Building a library of fixtures based on API documentation is one way to set up testing using `with_mock_API`. `httptest` also provides tools for collecting real HTTP responses to use as test fixtures. `capture_requests` is a context that records the responses from requests you make and stores them as mock files. This enables you to perform a series
of requests against a live server once and then build your test suite using
those mocks, running your tests in `with_mock_API`.

In an interactive session, it may be easier to use the functions `start_capturing` and `stop_capturing` rather than the context manager. You can set up your R session, call `start_capturing()`, and then do whatever commands or function calls that would make HTTP requests, and the responses will be grabbed.

Both the `capture_requests` context and the `start_capturing` function take a "path" argument, which lets you specify a location other than the current working directory to which to write the response files, and a "simplify" argument that, when `TRUE` (the default), it records simplified .json files where appropriate (200 OK response with `Content-Type: application/json`) and .R full "response" objects otherwise.

While recording requests to use later in tests can be very convenient, we don't always want to use captured requests, or at least not blindly. Real requests may contain information you want to sanitize or redact, like usernames, emails, or tokens. And real requests may be too big or messy to want to deal with. You may personally have more records or entities on the server than you'd want or need to include in mocks, so you can pare back that list of 80 entries that a query returns down to four or five and still have enough to test with.

# Mocks are text files

`httptest` stores these API mocks as plain-text files, which has several nice features, particularly relative to the alternative of storing serialized (binary) R objects. By storing fixtures as text files, you can more easily confirm that your mocks look correct, and you can more easily maintain them without having to re-record them. When you do edit them, text files are more easily handled by version-control systems like Git and Mercurial. Plain-text files can also have comments, so you can make notes as to why a certain fixture exists, what a particular value means, and so on, which will help the users of your package--and your future self!

By having mocks in human-readable text files, you can more easily extend your code. APIs are a data contract: if you give me this, I'll give you this back. At the same time, APIs are living things that evolve over time, and your code that communicates with an API needs to be able to change with them. If the API changes subtly, such as when adding an additional attribute to an object, you can just touch up the mocks. In addition, you can also ensure a degree of future-proofing of your code by tweaking a fixture file. For example, in [this fixture](https://github.com/Crunch-io/rcrunch/blob/master/inst/app.crunch.io/api/datasets/1.json#L33) in the `crunch` package, there's an extra nonsense attribute in the JSON, which is there just to ensure that the code doesn't break if there are new, unknown features added to the API response. That way, if the API grows new features, people who are using your package don't get errors if they haven't updated their version of the package to the new code that recognizes the new feature.

If you're responsible for the API as well as the R client code, the plain-text mocks can be a valuable source of documentation. Indeed, the file-system tree view of the mock files gives a visual representation of your API. For example, in the [crunch](https://github.com/Crunch-io/rcrunch/) package, the mocks show an API of catalogs that contain entities that may contain other subdocuments:

    app.crunch.io/
    ├── api
    │   ├── accounts
    │   │   ├── account1
    │   │   │   └── users.json
    │   │   └── account1.json
    │   ├── datasets
    │   │   ├── 1
    │   │   │   ├── export.json
    │   │   │   ├── filters
    │   │   │   │   └── filter1.json
    │   │   │   ├── filters.json
    │   │   │   ├── permissions.json
    │   │   │   ├── summary-73a614.json
    │   │   │   ├── variables
    │   │   │   │   ├── birthyr
    │   │   │   │   │   ├── summary-73a614.json
    │   │   │   │   │   └── values-3d4982.json
    │   │   │   │   ├── birthyr.json
    │   │   │   │   ├── gender
    │   │   │   │   │   ├── summary.json
    │   │   │   │   │   └── values-51980f.json
    │   │   │   │   ├── gender.json
    │   │   │   │   ├── starttime
    │   │   │   │   │   └── values-3d4982.json
    │   │   │   │   ├── starttime.json
    │   │   │   │   ├── textVar
    │   │   │   │   │   └── values-641ef3.json
    │   │   │   │   ├── textVar.json
    │   │   │   │   └── weights.json
    │   │   │   ├── variables-d118fa.json
    │   │   │   └── variables.json
    │   │   ├── 1.json
    │   │   └── search-c89aba.json
    │   └── users.json
    └── api.json

# Testing that requests aren't made

Mocking API responses isn't the only thing you might want to do in order to test your code. Sometimes, the request that matters is the one you don't make. `httptest` provides several tools to test requests without concern for the responses, as well as the ability to ensure that requests aren't made when they shouldn't be.

`without_internet` simulates the situation when any network request will
fail, as in when you are without an internet connection. Any HTTP request
through the verb functions in `httr`, or `utils::download.file`, will raise
an error. The error message raised has a well-defined shape, made of three
elements, separated by space: (1) the request
method (e.g. "GET", or for downloading, "DOWNLOAD"); (2) the request URL; and
(3) the request body, if present. The verb-expectation functions,
such as `expect_GET` and `expect_POST`, look for this shape.

Here's a example of how `without_internet` can be used to assert that code that should not make network requests in fact does not. This is a simplified version of a test from the [httpcache](https://github.com/nealrichardson/httpcache) package, a library that implements a query cache for HTTP requests in R. The point of the query cache is that only the first time you make a certain GET request should it hit the remote API; subsequent requests should read from the cache and not make a request. The test first makes a request (artificially, using `with_fake_HTTP`) to prime the cache.

    with_fake_HTTP({
        test_that("Cache gets set on GET", {
            expect_length(cacheKeys(), 0)
            expect_GET(a <- GET("https://app.crunch.io/api/"),
                "https://app.crunch.io/api/")
            expect_length(cacheKeys(), 1)
            expect_identical(a, getCache("https://app.crunch.io/api/"))
        })
    })

Then, using `without_internet`, the test checks two things: first, that doing the same GET succeeds because it reads from cache; and second, that if you bypass the query cache, you get an error because you tried to make a network request.

    without_internet({
        test_that("When the cache is set, can read from it even with no connection", {
            expect_identical(GET("https://app.crunch.io/api/")$url,
                "https://app.crunch.io/api/")
        })
        test_that("But uncached() prevents reading from the cache", {
            expect_error(uncached(GET("https://app.crunch.io/api/")),
                "GET https://app.crunch.io/api/")
        })
    })

This tells us that our cache is working as expected: we can get results from cache and we don't make a (potentially expensive) network request more than once.

## Assert the shape of request payloads

Another use of `without_internet` is to make assertions about requests--method, URL, and payload--that should be made. You could do this with mocks and use `with_mock_API`, but sometimes it is more clear what you're testing if you focus only on the requests. One case is when the response itself isn't that interesting or informative that the request did the correct thing. For example, if you're testing a POST request that alters the state of something on the server and returns 204 No Content status on success, nothing in the response itself (which would be stored in the mock file) tells you that the request you made was shaped correctly--the response has no content. A more transparent, readable test would just assert that the POST request was made to the right URL and had the expected request body.

Likewise, sometimes it is more explicit to test that the right request is made. In this example from the [crunch](https://github.com/Crunch-io/rcrunch/) package, there is a catalog resource containing three entities, each of which has an "archived" attribute. The `is.archived` method returns the value of that attribute:

    test_that("is.archived", {
        expect_identical(is.archived(catalog), c(FALSE, TRUE, FALSE))
    })

When we update the attributes of the catalog's entities, we send a PATCH request, and we only want to send values that are changing. That way, we don't unintentionally collide with any other concurrent actions happening on the server.

    test_that("'archive' sets the archived attribute and only PATCHes changes", {
        expect_PATCH(archive(catalog[2:3]),
            'https://app.crunch.io/api/datasets/',
            '{"https://app.crunch.io/api/datasets/3/":{"archived":true}}')
    })

The resulting state of the system (assuming no concurrent access) is the same whether the smaller PATCH request is sent or whether the overly verbose one is sent. If you've written logic in your R code to ensure that the smaller PATCH is sent, testing the shape of the request being made is the clearest way to demonstrate and assert that the expected behavior is happening.

A third instance of when you might care more about request body shape rather than the resulting response is when there are multiple paths in your R code that should lead to the same request being made. If you can assert that all of those variations result in the same request, then when it comes to testing the response and how your code handles it, you can do that once and not have to repeat for all of the input variations. This is particularly useful in conjunction with integration tests that run against a live server because it means you can have the same test coverage with fewer integration tests.

An example of this from the `crunch` package is in testing a `join` function, which has a similar syntax to the base R `merge` function. `merge` takes "by.x" and "by.y" arguments, which point to the variables in the "x" and "y" data.frames on which to match the rows when merging. It has a shortcut for the case where the variable have the same names in both data.frame, in which case you can just specify a "by" argument. To test that all of those combinations of specifying join keys result in the same request, the test defines a payload string to reuse

    testPayload <- paste0(
        '{"https://app.crunch.io/api/datasets/1/joins/95c0b45fe0af492594863f818cb913d2/":',
        '{"left_key":"https://app.crunch.io/api/datasets/1/variables/birthyr/",',
        '"right_key":"https://app.crunch.io/api/datasets/3/variables/birthyr/"}}')

and then asserts that three different ways of calling `join` result in the same `PATCH` request being made

    test_that("Can specify 'by' variables several ways", {
        expect_PATCH(join(ds1, ds2, by.x=ds1$birthyr, ds2$birthyr),
            'https://app.crunch.io/api/datasets/1/joins/',
            testPayload)
        expect_PATCH(join(ds1, ds2, by.x="birthyr", by.y="birthyr"),
            'https://app.crunch.io/api/datasets/1/joins/',
            testPayload)
        expect_PATCH(join(ds1, ds2, by="birthyr"),
            'https://app.crunch.io/api/datasets/1/joins/',
            testPayload)
    })

Subsequent integration tests that assert that the dataset is correctly modified on the server by `join` then only test with one of those ways of specifying the "by" variables. The R code that constructs the request is fully covered by these assertions.

Of course, you don't have to use `without_internet` to be able to assert about request shapes: you can get this same kind of behavior using `with_mock_API` if you make sure that you don't have a mock file corresponding to the request you want to test this way. When `with_mock_API` doesn't find a fixture file matching a request, it raises the same error that `without_internet` does. So, you can mix and match test strategies within the same context.

# Just test it

The goal of `httptest` is to remove a big obstacle to testing code that communicates with HTTP services: the HTTP service itself. If [`httr` makes HTTP easy](https://github.com/hadley/httr/blob/master/R/httr.r#L1) and [`testthat` makes testing fun](https://github.com/hadley/testthat/blob/master/R/test-that.R#L171), `httptest` makes testing your code that uses HTTP a simple pleasure.