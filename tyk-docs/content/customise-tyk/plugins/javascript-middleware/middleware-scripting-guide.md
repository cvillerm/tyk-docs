---
date: 2017-03-24T14:51:42Z
title: Middleware Scripting Guide
menu:
  main:
    parent: "Javascript Middleware"
weight: 0 
---

## Middleware Scripting

Middleware scripting is done in either a *pre* or *post* middleware chain context, dynamic middleware can be applied to both session-based APIs and Open (Keyless) APIs.

The difference between the middleware types are:

1.  **Pre**: These middleware instances do not have access to the session object (as it has not been created yet) and therefore cannot perform modification actions on them.

2.  **Post**: These middleware components have access to the session object (the user quota, allowances and auth data), but have the option to disable it, as deserialising it into the JSVM is computationally expensive and can add latency.

It is important to note that a new JSVM instance is created for *each* API that is managed, this means that inter-API communication is not possible via shared methods (they have different bounds), however it *is* possible using the session object if a key is shared across APIs.

### Enable the JSVM

Before you can use Javascript Middleware you will need to enable the JSVM

You can do this by setting `enable_jsvm` to `true` in your `tyk.conf` file.

#### Creating a middleware component

Tyk injects a `TykJS` namespace into the JSVM, this namespace can be used to initialise a new middleware component. Each middleware component should be in its own `*.js` file.

Creating a middleware object is done my calling the `TykJS.TykMiddleware.NewMiddleware({})` constructor with an empty object and then initialising it with your function using the `NewProcessRequest()` closure syntax.

Here is an example implementation:

```
    /* --- sample.js --- */
    
    // Create your middleware object
    var sampleMiddleware = new TykJS.TykMiddleware.NewMiddleware({});
    
    // Initialise it with your functionality by passing a closure that accepts two objects
    // into the NewProcessRequest() function:
    sampleMiddleware.NewProcessRequest(function(request, session) {
    
        console.log("This middleware does nothing, but will print this to your terminal.")
    
        // You MUST return both the request and session metadata    
        return sampleMiddleware.ReturnData(request, session.meta_data);
    });    
```

#### Middleware component variables

As well as the API functions that all JSVM components share, the middleware components have access to some data structures that are performant and allow for the modification of both the request itself and the session. These objects are exposed to the middleware in the form of the `request` and `session` objects in the `NewProcessRequest(function(request, session) {};` call.

In the example above, we can see that we return these variables - this is a requirement, and omitting it can cause the middleware to fail, this line should be called at the end of each process:

```
    return sampleMiddleware.ReturnData(request, session.meta_data);
```

This allows the middleware machinery to perform the necessary writes and changes to the two main context objects.

#### The `request` object

The `request` object provides a set of arrays that can be manipulated, that when changed, will affect the request as it passes through the middleware pipeline, the `request` object looks like this:

```
    {
        Headers       map[string][]string
        SetHeaders    map[string]string
        DeleteHeaders []string
        Body          string
        URL           string
        AddParams     map[string]string
        DeleteParams  []string
        ReturnOverrides {
            ResponseCode: int
            ResponseError: string
        }
    }
```

*   `Headers`: This is an object of string arrays, and represents the current state of the request header. This object cannot be modified directly, but can be used to read header data.
*   `SetHeaders`: This is a key-value map that will be set in the header when the middleware returns the object, existing headers will be overwritten and new headers will be added.
*   `DeleteHeaders`: Any header name that is in this list will be deleted from the outgoing request. `DeleteHeaders` happens before `SetHeaders`.
*   `Body`: This represents the body of the request, if you modify this field it will overwrite the request.
*   `URL`: This represents the path portion of the outbound URL, use this to redirect a URL to a different endpoint upstream.
*   `AddParams`: You can add parameters to your request here, for example internal data headers that are only relevant to your network setup.
*   `DeleteParams`: These parameters will be removed from the request as they pass through the middleware. `DeleteParams` happens before `AddParams`.
*   `ReturnOverrides`: Values stored here are used to stop or halt middleware execution and return an error code if the middleware operation has failed.

Using the methods outlined above, alongside the API functions that are made available to the VM, allows for a powerful set of tools for shaping and structuring inbound traffic to your API, as well as processing, validating or re-structuring the data as it is inbound.

#### The `session` object

Tyk uses an internal session representation to handle the quota, rate limits, and access allowances of a specific key. This data can be made available to POST-processing middleware for processing. the session object itself cannot be edited, as it is crucial to the correct functioning of Tyk.

In order for middleware to be able to transfer data between each other, the session object makes available a `meta_data` key/value field that is written back to the session store (and can be retrieved by the middleware down the line) - this data is permanent, and can also be retrieved by the REST API from outside of Tyk using the `/tyk/keys/` method.

The session object has the same representation as the one used by the API:

```
    {
        "allowance": 999,
        "rate": 1000,
        "per": 60,
        "expires": 0,
        "quota_max": -1,
        "quota_renews": 1406121006,
        "quota_remaining": 0,
        "quota_renewal_rate": 60,
        "access_rights": {
            "234a71b4c2274e5a57610fe48cdedf40": {
                "api_name": "Versioned API",
                "api_id": "234a71b4c2274e5a57610fe48cdedf40",
                "versions": [
                    "v1"
                ]
            }
        },
        "org_id": "53ac07777cbb8c2d53000002",
        "meta_data": {
            "your-key": "your-value"
        }
    }
```

There are other ways of accessing and editing a session object by using the Tyk JSVM API functions.
