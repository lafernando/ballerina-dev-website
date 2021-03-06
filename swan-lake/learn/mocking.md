---
layout: ballerina-left-nav-pages-swanlake
title: Mocking
description: Learn how to use Ballerina's built-in mocking API provided by the test to test modules
 independent from other modules and external endpoints.
keywords: ballerina, programming language, testing, mocking
permalink: /swan-lake/learn/mocking/
active: mocking
redirect_from:
  - /swan-lake/learn/mocking
---

# Mocking

Mocking is useful to control the behavior of functions and objects to control the communication with other modules and external endpoints. A mock can be created by defining return values or replacing the entire object or function with a user-defined equivalent. This feature will help you to test the Ballerina code independently from other modules and external endpoints.


## Mocking Objects

The test module provides capabilities to mock an object for unit testing. This allows you to control the behavior of object member functions and values of member fields via stubbing or replacing the entire object with a user-defined equivalent. This feature will help you to test the Ballerina code independently from other modules and external endpoints.

Mocking of objects can be done in two ways.

1. Creating a test double (providing an equivalent object in place of the real)
2. Stubbing the member function or member variable (specifying the behavior of functions and values of variables)


### Creating a test double

You can write a custom mock object and substitute it in place of the real object. The custom object should be made structurally equivalent to the real object via the mocking features in the test module.

***Example:***

Let's make changes to the example in the [Quick Start](/swan-lake/learn/testing-quick-start) page. In order to test the
 `getRandomJoke` function without actually calling the endpoint, use the `test:mock` function to mock the `get` remote function of the client object.

Change the content of the ***main_test.bal*** file as follows:

```ballerina
// main_test.bal
import ballerina/test;
import ballerina/http;

// This is the object to be used as the test double for any `http:Client` object. 
// Only the `get` function is implemented since it is the only function used in the sample.
public type MockHttpClient client object {
    public remote function get(@untainted string path, public http:RequestMessage message = ()) 
    	returns http:Response|http:ClientError {

        http:Response res = new;
        res.statusCode = 500;
        return res;
    }
};

// This test function tests the behavior of the `getRandomJoke` function when the API returns an error response
@test:Config {}
public function testTestDouble() {
    // create and assign a test double to the HTTP object `clientEndpoint` object
    clientEndpoint=<http:Client>test:mock(http:Client, new MockHttpClient());
    // invoke function to test
    string|error result = getRandomJoke("Sheldon");
    // verify that the function returns an error
    test:assertTrue(result is error);
}
```

### Stubbing member functions and variables of an object

Instead of creating a test double, you may also choose to create a default mock object and stub the functions to return a specific value or to do nothing.


#### Stubbing to return a specific value

***Example:***
The example in the [Quick Start](/swan-lake/learn/testing-quick-start) page shows how the `get` function of the client object can be
 stubbed to return a value. Let’s make changes to that example.

**main.bal**

```ballerina
// main.bal

import ballerina/io;
import ballerina/http;
import ballerina/stringutils;

http:Client clientEndpoint = new("https://api.chucknorris.io/jokes/");

// This function performs a `get` request to the Chuck Norris API and returns a random joke 
// from the provided category with the name replaced with the provided input or an error 
// if API invocations fail.
function getRandomJoke(string name, string category = "food") returns string|error {
    http:Response response = checkpanic clientEndpoint->get("/categories");
    if (response.statusCode == http:STATUS_OK) {
        json[] categories = <json[]>response.getJsonPayload();
        if (!isCategoryAvailable(categories, category)) {
            error err = error("'"+category+"' is not a valid category.");
            io:println(err.message());
            return err;
        }
    } else {
    	return createErrorResponse(response);
    }
    response = checkpanic clientEndpoint->get("/random?category=" + category);
    if (response.statusCode == http:STATUS_OK) {
        json payload = <json>response.getJsonPayload();
        json joke = <json>payload.value;
        string replacedText = stringutils:replace(joke.toJsonString(), "Chuck Norris", name);
        return replacedText;
    } else {
    	return createErrorResponse(response);
    }
}

// This function checks if the provided category is a valid one
function isCategoryAvailable(json[] categories, string category) returns boolean {
    foreach var cat in categories {
        if (cat.toJsonString() == category) {
            return true;
        }
    }
    return false;
}

// Returns an error based on the HTTP response
function createErrorResponse(http:Response response) returns error {
    error err = error("error occurred while sending GET request");
    io:println(err.message(), ", status code: ", response.statusCode);
    return err;
}
```
**main_test.bal**

The test functions in this file shows different options to stub the behavior of the `get` function of the HTTP Client in
 order to test the `getRandomJoke` function.
```ballerina
// main_test.bal
import ballerina/test;
import ballerina/http;

// This test stubs the behavior of the `get` function to return a specified value
// for testing the `getRandomJoke` function
@test:Config {}
public function testReturn() {
    // create a default mock HTTP Client and assign it to the `clientEndpoint` object
    clientEndpoint = <http:Client>test:mock(http:Client);
    // stub to return the specified mock response when `get` function is called
    test:prepare(clientEndpoint).when("get").thenReturn(getMockResponse());
    // stub to return the specified mock response when the specified argument is passed
    test:prepare(clientEndpoint).when("get").withArguments("/categories").thenReturn(getCategoriesResponse());
    // invoke the function to test
    string result = checkpanic getRandomJoke("Sheldon");
    // verify the return value against the expected string
    test:assertEquals(result, "When Sheldon wants an egg, he cracks open a chicken.");
}

// This test stubs the behavior of the `get` function to return a specified sequence of values
// for each `get` function invocation when testing the `getRandomJoke` function
@test:Config {}
public function testReturnSequence() {
    // create a default mock HTTP Client and assign it to the `clientEndpoint` object
    clientEndpoint = <http:Client>test:mock(http:Client);
    // stub to return the corresponding value for each invocation i.e., the first call to `get` will return the mock
    // response containing categories and the second call will return the mock response containing the joke
    test:prepare(clientEndpoint).when("get").thenReturnSequence(getCategoriesResponse(), getMockResponse());
    // invoke the function to test
    string result = checkpanic getRandomJoke("Sheldon");
    // verify the return value against the expected string
    test:assertEquals(result, "When Sheldon wants an egg, he cracks open a chicken.");
}

// This function shows how to stub a member variable value of the HTTP Client object
@test:Config {}
function testMemberVariable() {
    // create a default mock HTTP Client and assign it to the `clientEndpoint` object
    clientEndpoint = <http:Client>test:mock(http:Client);
    // stub the value of the `url` variable to return the specified string
    test:prepare(clientEndpoint).getMember("url").thenReturn("https://foo.com/");
    // verify if the specified value is set
    test:assertEquals(clientEndpoint.url, "https://foo.com/");
}

// Returns a mock HTTP response to be used for random joke API invocation
function getMockResponse() returns http:Response {
    http:Response mockResponse = new;
    json mockPayload = {"value":"When Chuck Norris wants an egg, he cracks open a chicken."};
    mockResponse.setPayload(mockPayload);
    return mockResponse;
}

// Returns a mock response to be used for the category API invocation
function getCategoriesResponse() returns http:Response {
    http:Response categoriesRes = new;
    json[] payload = ["animal","food","history","money","movie"];
    categoriesRes.setJsonPayload(payload);
    return categoriesRes;
}
```

#### Stubbing to do nothing

***Example:***

**main.bal**

```ballerina
// main.bal
import ballerina/email;
import ballerina/io;

email:SmtpClient smtpClient = new ("localhost", "admin","admin");
// This function sends out emails to specified email addresses and returns an error if sending failed.
function sendNotification(string[] emailIds) returns error? {
    email:Email msg = {
        'from: "builder@abc.com",
        subject: "Error Alert ...",
        to: emailIds,
        body: ""
    };
    email:Error? response = smtpClient->send(msg);
    if (response is error) {
	io:println("error while sending the email: " + response.message());
  	return response;
    }
}
```
**main_test.bal**

```ballerina
// main_test.bal
import ballerina/test;
import ballerina/email;

// This function stubs the `send` function of the `SmtpClient` client object to do nothing when invoked
// for testing the `sendNotification` function
@test:Config {}
function testSendNotification() {
    // create a default mock SMTP client and assign it to the `smtpClient` object
    smtpClient = <email:SmtpClient>test:mock(email:SmtpClient);
    // stub to do nothing when the`send` function is invoked
    test:prepare(smtpClient).when("send").doNothing();
    string[] emailIds = ["user1@test.com", "user2@test.com"];
    // invoke the function to test and verify that no error occurred
    test:assertEquals(sendNotification(emailIds), ());
}
```

## Mocking Functions

Ballerina test framework provides the capability to mock a function. Using the mocking feature, you can easily mock a function in a module that you are testing or a function of an imported module. This feature will help you to test your Ballerina code independently from other modules and functions.


### Mocking an imported function

The function specified with the `@test:Mock {}` annotation will be considered as a mock function that gets triggered every time the original function is called. The original function that will be mocked should be defined using the following annotation value fields.

*   ***moduleName : "&lt;moduleName&gt;"*** - Name of the module in which the function to be mocked resides in. If the
 function is within the same module, this can be left blank or “.” (no module) can be passed. If the function is in a different module but within the same project, just passing the module name will suffice. For functions in completely separate modules, the fully-qualified module name must be passed, which includes the `orgName` and the `version` (i.e., `orgName/module:version`). For native functions, the Ballerina module needs to be specified.
*   ***functionName : "&lt;functionName&gt;"*** - Name of the function to be mocked.

Mocking an imported function will apply the mocked function to every instance of the original function call. It is not limited to the test the file, which is being mocked. 

***Example:***

**main.bal**

```ballerina
// main.bal
import ballerina/io;
import ballerina/math;

// The function prints the value of PI using the `io:println` function
public function printMathConsts() {
   io:println("Value of PI : ", math:PI);
}
```

**main_test.bal**

```ballerina
// main.test.bal
import ballerina/test;

(any|error)[] outputs = [];

// This is the mock function, which replaces the `io:println` function
@test:Mock {
    moduleName: "ballerina/io",
    functionName: "println"
}
function mockIoPrintLn((any|error)... text) {
    // Append the print statement to a global array
    outputs.push(text);
}

// This function tests the PI constant value provided by the `ballerina/math` module
@test:Config {}
function testMathConsts() {
    // Invoke the function to test
    printMathConsts();
    // Verify the value provided by the math module against the expected value
    test:assertEquals(outputs[0].toString(), "Value of PI :  3.141592653589793");
}
```

#### Mocking a function in the same module

The object specified with the `@test:MockFn {}` annotation will be considered as a mock function that gets triggered
 every time the original function is called. Subsequent to the declaration, the function call should be stubbed using the available function mocking features. Different behaviors can be defined for different test cases if required.

***Example:***

**main.bal**

```ballerina
// main.bal

// This function returns the result provided by the `intAdd` function
public function addValues(int a, int b) returns int {
    return intAdd(a, b);
}

// This function adds two integers and returns the result
public function intAdd(int a, int b) returns int {
    return (a + b);
}
```

**main_test.bal**

The test functions in this file shows different options to stub the behavior of a function written in the module that
 is under test.

```ballerina
// main_test.bal
import ballerina/test;

// This is the initialization of the mock function that should be called in place of the `intAdd` function
@test:MockFn { functionName: "intAdd" }
test:MockFunction intAddMockFn = new();

// This test stubs the behavior of the `intAdd` function to return a specific value
// for testing the `addValues` function
@test:Config {}
function testReturn() {
    // stub to return the specified value when the `intAdd` is invoked
    test:when(intAddMockFn).thenReturn(20);
    // stub to return the specified value when the `intAdd` is invoked with the specified arguments
    test:when(intAddMockFn).withArguments(0, 0).thenReturn(-1);
    
    test:assertEquals(addValues(10, 6), 20, msg = "function mocking failed");
    test:assertEquals(addValues(0, 0), -1, msg = "function mocking with arguments failed");
}

// This test stubs the behavior of the `intAdd` function to call another function
// for testing the `addValues` function
@test:Config {}
function testCall() {
    // stub to call another function when `intAdd` is called
    test:when(intAddMockFn).call("mockIntAdd");
    test:assertEquals(addValues(11, 6), 5, msg = "function mocking failed");
}

// Mock function to be used in place of the `intAdd` function
public function mockIntAdd(int a, int b) returns int {
    return (a - b);
}
```

## What's Next
 
Now, that you are aware of the details on writing tests, learn the different options that can be used when [Executing
 Tests](/swan-lake/learn/executing-tests).
