# ussd-menu-builder

[![Build Status](https://travis-ci.org/habbes/ussd-menu-builder.svg?branch=master)](https://travis-ci.org/habbes/ussd-menu-builder.svg?branch=master)


Easily compose USSD menus in Node.Js, compatible with 
[Africastalking API](https://africastalking.com).

# QuikStart

## Installation

```
$ npm install ussd-menu-builder
```

## Features
- Use intuitive states to compose USSD menus
- Makes it easier to build complex nested menus
- Use simple input matching or regular expressions, custom asynchronous
functions to resolve routes from one state to another
- The state-based approach allows you to easily modularize complex menus
in different files

## Quick Example

```javascript
const UssdMenu = require('ussd-menu-builder');
let menu = new UssdMenu();

// Define menu states
menu.startState({
    run: () => {
        // use menu.con() to send response without terminating session      
        menu.con('Welcome. Choose option:' +
            '\n1. Show Balance' +
            '\n2. Buy Airtime');
    },
    // next object links to next state based on user input
    next: {
        '1': 'showBalance',
        '2': 'buyAirtime'
    }
});

menu.state('showBalance', {
    run: () => {
        // fetch balance
        fetchBalance(menu.args.phoneNumber).then(function(bal){
            // use menu.end() to send response and terminate session
            menu.end('Your balance is KES ' + bal);
        });
    }
});

menu.state('buyAirtime', {
    run: () => {
        menu.con('Enter amount:');
    },
    next: {
        // using regex to match user input to next state
        '*\\d+': 'buyAirtime.amount'
    }
});

// nesting states
menu.state('buyAirtime.amount', {
    run: () => {
        // use menu.val to access user input value
        var amount = Number(menu.val);
        buyAirtime(menu.args.phoneNumber, amount).then(function(res){
            menu.end('Airtime bought successfully.');
        });
    }
});

// Registering USSD handler with Express

app.post('/ussd', function(req, res){
    menu.run(req.body, ussdResult => {
        res.send(ussdResult);
    });
});

```

# Guide
## Introduction
The USSD Menu Builder uses a state machine to create a USSD menu. A state
is created for each menu. Each state has a unique name and a set of rules
used to link to other states based on the user input.

### Creating a menu
Before you can create any states, you first need to create an instance of
the menu.

```javascript
const UssdMenu = require('ussd-menu-builder');
const menu = new UssdMenu();
```

### Running the menu
The **```menu.run(args, resultCallback)```** goes through the menu and finds
the appropriate state to run based on the user input.

Thar **```args```** object should contain the following keys coming from
the  [Africastalking API](https://africastalking.com):

- **`sessionId`**: unique session ID that persists through the entire USSD session,
can be used to store temporary that may be retrieved from different states
during the session
- **`serviceCode`**: the USSD code registered with your serviceCode
- **`phoneNumber`**: the end user's phone Number
- **`text`**: The raw USSD input. It has the following format ```1*2*4*1```:
a string containing the input at each hop, separated by the asterisk symbol (```*```).
This is parsed by the ```UssdMenu``` to find the appropriate state to run at each hop.

After the matched state runs, the resultCallback is called with the response from the state.

Here's an example registering a handler with the [express](https://expressjs.com) framework:
```javascript

app.post('/ussd', (req, res) => {
    let args = {
        phoneNumber: req.body.phoneNumber,
        sessionId: req.body.sessionId,
        serviceCode: req.body.serviceCode,
        text: req.body.text
    };
    menu.run(args, resMsg => {
        res.send(resMsg);
    });
})

```

## Defining states

The **`menu.state(name, options)`** method is used to define states. I takes the name of the state
and an object with the following properites:
- **`run`**: a function that's called when the state is resolved
- **`next` (optional)**: an object that contains rules of how to match the input of this state
to other states. *This is not required for final states*.
- **`defaultNext` (optional)**: the name of the state to default to if the user
input could not be matched by the rules defined in the `next` object.
If not provided, the same state will be used as a fallback i.e. the same menu will
be displayed to the user.

Here's an example:
```javascript
menu.state('stateName', {
    run: function(){
        menu.con('Choose Option' +
            '\n1. Load Account' +
            '\n2. View Catalogue' +
            '\n3. Check Balance'
        );
    },
    next: {
        '1': 'loadAccount',
        '2': 'catalogue',
        '3': 'balance'
    },
    defaultNext: 'invalidOption'
});
```

### The **`run`** function
Each state defines it's own `run` method which is called when that
state is matched. This is where you should place the logic for a given
state.

#### Retrieving user input
Use **```menu.val```** property to access the current user input.

#### Accessing ussd parameters
You can access the ussd parameters through the **```menu.args```** object.
This parameters should come from the API Gateway and are passed to the
```menu.run``` method.

#### Sending the response
You must use either (not both) of the two methods to send
a response to be displayed to the user:
- **`menu.con(msg)`**: Sends the result to be displayed to the user without
terminating the session i.e. the user can reply with further input.
- **`menu.end(msg)`**: Sends the response to be displayed to the user and
requests the session to be terminated i.e. the user cannot provide further
input. **Note:** *This consequently makes the state a final state and therefore the
```next``` object does not need to be defined*

Example:

```javascript
menu.state('thisState', {
    run: function(){
        let value = menu.val;
        let session = getSession(menu.args.sessionId);
        let phone = menu.args.phoneNumber;
        session.set('phone', phone);
        session.set('value', value);
        menu.end('You entered: ' + value);
    }
});
```

### The Start State
This is the first state or first menu to be displayed by the user.
It is created using the **```menu.startState(options)```**. It uses
the reserved name ```'__start__'```.

``` javascript
menu.startState({
    run: function(){
        ...
    }
    next: {
        ...
    }
});
```

***
**Note**: the ```menu.state()``` and ```menu.startState()``` methods return
the same menu object instance for convenience.

```javascript
menu.startState({
    ...
})
.state('state1', {
    ...
})
.state('state2', {
    ...
})
```
***

### Matching States
To link states you use the ```next``` object to map user input to a state name.
You can match input directly by value or with a regular expression.

#### Matching direct values
Simply add the expected string value as a key in the next object.

#### Matching with regular expressions
Begin the key with an asterisk (**```*```**) to indicate that the key should
be treated like a regular expression e.g. ```'*\\[a-zA-Z]+'``` would match
any input containing only lowercase or uppercase letters.

Remember you can use ```menu.val``` in the matched state to retrieve the actual user input.

Example:
```javascript
menu.state('registration', {
    run: function(){
        menu.con('Enter your name');
    },
    next: {
        '*[a-zA-Z]+': 'registration.name'
    }
});

menu.state('registration.name', {
    run: function(){
        let name = menu.val;
        let session = getSession(menu.args.sessionId);
        session.set('name', name);
        menu.con('Enter your email');
    },
    next: {
        '*\\w+@\\w+\\.\\w+': 'registration.email'
    }
});
```

#### Matching with empty rule on Start State
If the start state does not define a ```run``` method, you provide
an empty string as key in ```next``` to redirect to another state.

```javascript
menu.startState({
    next: {
        '': function(){
            if(user){
                return 'userMenu';
            }
            else {
                return 'registerMenu';
            }
        }
    }
});
```
### Linking states
Beside mapping user input directly to a state name, you can map it to
a function with returns a state name, synchronously with a simple
return statement or asynchronously with a callback or a promise.

#### Mapping to a direct state name
```javascript
menu.state('thisState', {
    ...
    next: {
        'input': 'nextState'
    }
})
```

#### Mapping to a synchronous function
```javascript
menu.state('thisState', {
    ...
    next: {
        'input': function(){
            if(test){
                return 'nextState';
            } else {
                return 'otherState';
            }
        }
    }
});
```
#### Mapping to an async function with callback
```javascript
menu.state('thisState', {
    ...
    next: {
        'input': function(callback){
            runAsyncCode(function(err, res){
                if(res){
                    callback('nextState');
                } else {
                    callback('otherState');
                }
            })
        }
    }
});
```

#### Mapping to an async function with promise
```javascript
menu.state('thisState', {
    ...
    next: {
        'input': function(){
            return new Promise((resolve, reject) => {
                resolve('nextState');
            });
        }
    }
});
```

### Jumping to different state
You can jump to a different state from the ```run``` function of one
state using the **```menu.go(stateName)```** method. This effectively
breaks the state chain (subsequent states will not be reachable) 
and is therefore only useful if jumping to a final state.

```javascript
menu.state('thisState', {
    run: function(){
        menu.go('otherState');
    }
});

menu.state('otherState', {
    run: function(){
        menu.end('Thank you!');
    }
});
```

The **```menu.goStart()```** method can be used to jump to the start state
from within another state.

## Nesting states
The library treats a USSD menu like a chain of interlinked states and therefore
has not internal concept of nesting. However you can achieve complex menus
with nested submenus by linking states appropriately. In addition you
could use a naming convention of your choice to make it clearer to see how
states are related. In these examples I used the following convention of 
separating menu levels with a dot.