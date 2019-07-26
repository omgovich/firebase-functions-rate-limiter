# Firebase functions rate limiter
[![npm](https://img.shields.io/npm/v/firebase-functions-rate-limiter.svg?style=flat-square)](https://www.npmjs.com/package/firebase-functions-rate-limiter)  [![build](https://travis-ci.com/Jblew/firebase-functions-rate-limiter.svg?branch=master)](https://travis-ci.com/Jblew/firebase-functions-rate-limiter) [![License](https://img.shields.io/github/license/Jblew/firebase-functions-rate-limiter.svg?style=flat-square)](https://github.com/Jblew/firebase-functions-rate-limiter/blob/master/LICENSE) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)



Q: How to limit rate of firebase function calls?
A: Use `firebase-functions-rate-limiter`

Mission: **limit number of calls per specified period of time**



## Key features:

- Two backends: more efficient Realtime Database or Firestore for convenience?
- Easy: call single function, no configuration
- Efficient: only a single call read call to database (or firestore), two calls if limit not exceeded and usage is recorded
- Concurrent-safe: uses atomis transactions in both backends
- Clean: Uses only one key (or collection in firestore backend), creates single document for each qualifier. Does not leave rubbish in your database.
- Typescript typings included
- No firebase configuration required. You do not have to create any indexes or rules.



## Installation

```bash
$ npm install --save firebase-functions-rate-limiter
```

Then:

```typescript
import FirebaseFunctionsRateLimiter from "firebase-functions-rate-limiter";
// or
const FirebaseFunctionsRateLimiter = require("firebase-functions-rate-limiter");
```



##Usage

**Example 1**: limit calls for everyone:

```javascript
import * as admin from "firebase-admin";
import * as functions from "firebase-functions";
import { FirebaseFunctionsRateLimiter } from "firebase-functions-rate-limiter";

admin.initializeApp(functions.config().firebase);
const database = admin.database();

const limiter = FirebaseFunctionsRateLimiter.withRealtimeDbBackend(
    {
        name: "rate_limiter_collection",
        maxCalls: 2,
        periodSeconds: 15,
    },
    database,
);
exports.testRateLimiter = 
  functions.https.onRequest(async (req, res) => {
    await limiter.rejectOnQuotaExceeded(); // will throw HttpsException with proper warning

    res.send("Function called");
});

```

>  You can use two functions: `limiter.rejectOnQuotaExceeded(qualifier?)` will throw an *functions.https.HttpsException* when limit is exceeded while `limiter.isQuotaExceeded(qualifier?)` gives you the ability to choose how to handle the situation.


**Example 2**: limit calls for each user separately (function called directly - please refer [firebase docs on this topic](https://firebase.google.com/docs/functions/callable)):

```javascript
import * as admin from "firebase-admin";
import * as functions from "firebase-functions";
import { FirebaseFunctionsRateLimiter } from "firebase-functions-rate-limiter";

admin.initializeApp(functions.config().firebase);
const database = admin.database();

const perUserlimiter = FirebaseFunctionsRateLimiter.withRealtimeDbBackend(
    {
        name: "per_user_limiter",
        maxCalls: 2,
        periodSeconds: 15,
    },
    database,
);

exports.authenticatedFunction = 
  functions.https.onCall(async (data, context) => {
    if (!context.auth || !context.auth.uid) {
        throw new functions.https.HttpsError(
            "failed-precondition",
            "Please authenticate",
        );
    }
    const uidQualifier = "u_" + context.auth.uid;
    const isQuotaExceeded = await perUserlimiter.isQuotaExceeded(uidQualifier);
    if (isQuotaExceeded) {
        throw new functions.https.HttpsError(
            "failed-precondition",
            "Call quota exceeded for this user. Try again later",
        );
    }
  
    return { result: "Function called" };
});

```



### Step-by-step

**#1** Initialize admin app and get Realtime database object

```typescript
admin.initializeApp(functions.config().firebase);
const database = admin.database();
```

**#2** Create limiter object outside of the function scope and pass the configuration and Database object. Configuration options are listed below.

```typescript
const someLimiter = FirebaseFunctionsRateLimiter.withRealtimeDbBackend(
    {
        name: "limiter_some",
        maxCalls: 10,
        periodSeconds: 60,
    },
    database,
);
```

**#3** Inside the function call isQuotaExceeded. This is an async function so not forget about **await**! The function will check if the limit was exceeded. If limit was not exceeded it will record this usage and return true. Otherwise, write will be only called if there are usage records that are older than the specified period and are about to being cleared.

```typescript
exports.testRateLimiter = 
  functions.https.onRequest(async (req, res) => {
    const quotaExceeded = await limiter.isQuotaExceeded();
    if (quotaExceeded) {
    	// respond with error
    } else {
      // continue
    }
```

**#3 with qualifier**. Optionally you can pass **a qualifier** to the function. A qualifier is a string that identifies a separate type of call. If you pass a qualifier, the limit will be recorded per each distinct qualifier and won't sum up.

```typescript
exports.testRateLimiter = 
  functions.https.onRequest(async (req, res) => {
    const qualifier = "user_1";
    const quotaExceeded = await limiter.isQuotaExceeded(qualifier);
    if (quotaExceeded) {
    	// respond with error
    } else {
      // continue
    }
```



## Configuration

```typescript
const configuration = {
  name: // a collection with this name will be created
  periodSeconds: // the length of test period in seconds
  maxCalls: // number of maximum allowed calls in the period
  debug: // boolean (default false)
};
```

#### Choose backend:

```typescript
const limiter = FirebaseFunctionsRateLimiter.withRealtimeDbBackend(configuration, database)
// or
const limiter = FirebaseFunctionsRateLimiter.withFirestoreBackend(configuration, firestore)
```





### Firebase configuration

There is no configuration needed in the firebase. This library does not do document search, so you do not need indexes. Also, functions are executed in the firebase admin environment, so you do not have to specify any rules.



### Need help?

- Feel free to email me at <jedrzej@lewandowski.doctor>



### Would like to help?

Warmly welcomed:

- Bug reports via issues
- Enhancement requests via via issues
- Pull requests
- Security reports to jedrzej@lewandowski.doctor



***

Made with ❤️ by [Jędrzej Lewandowski](https://jedrzej.lewandowski.doctor/).

