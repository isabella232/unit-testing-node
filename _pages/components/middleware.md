---
title: Middleware class
---
The `Middleware` class is where all of the pieces we've built up so far get
integrated. It is the figurative brain and heart of the application. The name
refers to its role in implementing a piece of [Hubot receive
middleware](https://hubot.github.com/docs/scripting/#middleware). You can find
it in the
[`exercise/lib/middleware.js`]({{ site.baseurl }}/exercise/lib/middleware.js)
file.

If you've skipped to this chapter, you can establish the starting state of the
`exercise/` files for this chapter by running:

```sh
$ ./go set-middleware
```

## What to expect

Thanks to the work we've done encapsulating configuration validation in the
`Config` class, rule matching logic in the `Rule` class, and API calls to
Slack and GitHub in the `SlackClient` and `GitHubClient` classes, `Middleware`
can focus squarely on the core application logic.

We also don't need to test all possible corner and error cases for those other
components, since they have been thoroughly tested in isolation. In fact, we
don't need to run any HTTP servers in the `Middleware` test, because we can
use [test
doubles](http://googletesting.blogspot.com/2013/07/testing-on-toilet-know-your-test-doubles.html)
to simulate their behavior. This will make the `Middleware` tests easier to
write, to maintain, and to understand.

In short, we will learn to:

- build our core application object using [composition rather than
  inheritance]({{ site.baseurl }}/concepts/object-composition-vs-inheritance)
- write more controllable, maintainable, readable tests using test doubles and
  a technique known as [dependency
  injection]({{ site.baseurl }}/concepts/dependency-injection)
- using the [`sinon` library](http://sinonjs.org/) to create
  [test doubles](http://googletesting.blogspot.com/2013/07/testing-on-toilet-know-your-test-doubles.html)
- learn how to use `Promises` with mocha and chai

## The core algorithm

`Middleware` implements the core algorithm of the application:

- Match an incoming `reaction_added` message against the rules.
- If a match is found, get a list of all of the reactions to the message.
- If there is no "success" reaction (defined in the configuration), file a
  GitHub issue.
- Add a "success" reaction to the message.
- Post the link to the GitHub issue in the channel containing the message.

Thanks to the other classes we've written, `Middleware` is not concerned with
the details handled by those classes. This makes the `Middleware` class itself
easier to implement, to understand, and especially easier to test.

## Starting to build `Middleware`

The beginning of the `middleware.js` file, where the `Middleware` constructor
is defined, looks like this:

```js
/* jshint node: true */

'use strict';

module.exports = Middleware;

function Middleware(config, slackClient, githubClient) {
  this.rules = config.rules;
  this.successReaction = config.successReaction;
  this.slackClient = slackClient;
  this.githubClient = githubClient;
}

Middleware.prototype.execute = function(/* context, next, done */) {
};
```

There are two very important things to notice about the constructor. First,
the `rules`, `successReaction`, `slackClient`, and `githubClient` objects
become properties of the `Middleware` object. Rather than directly implement
configuration validation, rule matching, and HTTP request behavior, we
delegate handling of those detailed operations to each respective object.

Neither does `Middleware` inherit any behavior from other objects. Every
dependency on behavior not implemented directly by `Middleware` itself is made
explicit by a method call on a collaborating object. This is an illustration of
[object composition]({{ site.baseurl }}/concepts/object-composition-vs-inheritance),
one of the core design principles that leads to more readable, more maintable,
more testable code.

The second thing to notice is that the `Middleware` object is not creating
these collaborating objects itself. Rather, these _dependencies_ are
_injected_ into the object by the code creating the `Middleware` object. The
finished application will configure the `Middleware` to use real `Config`,
`SlackClient`, and `GitHubClient` objects. However, our tests can use
alternate implementations of `SlackClient` and `GitHubClient` in particular to
exercise `Middleware` behavior without making actual Slack and GitHub API
calls. [Dependency injection]({{ site.baseurl }}/concepts/dependency-injection)
relies upon object composition to make it even easier to write focused,
targeted, isolated, stable, readable, maintainable, valuable automated tests.

## Creating `Rule` objects from `config.rules`

The first thing we need to do is promote the flat JSON objects from
`config.rules` to full-fledged `Rule` objects. The `Config` object has already
ensured that `config.rules` contains valid `Rule` specifications, so all we
have to do is map each specification to a behavior-rich object:

```js
  this.rules = config.rules.map(function(rule) {
    return new Rule(rule);
  });
```

## Understanding the `execute(context, next, done)` function signature

The `Middleware` object implements the [Hubot receive middleware
specification](https://hubot.github.com/docs/scripting/#middleware), which
defines the following arguments:

- **context**: for [receive
  middleware](https://hubot.github.com/docs/scripting/#receive-middleware-api),
  this contains a `response` property that itself contains the incoming
  `message` and a `reply` method to send a response message to the user
- **next**: a callback function that runs the next piece of middleware; must
  be called with **done** as an argument, or a function that eventually calls
  **done**
- **done**: a callback function taking no arguments that signals completion of
  middleware processing; typically passed as the sole argument to **next**,
  but a middleware function can call it to abort further processing

The first thing we will do is uncomment the arguments and pick apart the parts
of `context` that we need into separate variables:

```js
Middleware.prototype.execute = function(context, next, done) {
  var response = context.response,
      message = response.message.rawMessage;
}
```

## The `reaction_added` message format

The `message` variable holds the raw JSON object representing the
[`reaction_added` message](https://api.slack.com/events/reaction_added). These
messages will look like this:

```json
{
  "type": "reaction_added",
  "user": "U024BE7LH",
  "item": {
    "type": "message",
    "channel": "C2147483705",
    "ts": "1360782804.083113"
  },
  "reaction": "thumbsup",
  "event_ts": "1360782804.083113"
}
```

Note that, per the API documentation the `item` member can also be a `file` or
a `file_comment`. Our implementation will not handle those types, though the
actual application may eventually add support for them.

The message receiving the message is uniquely identified by the channel ID
(`channel`) and timestamp (`ts`). This is why we added those parameters to our
`SlackClient` methods that implement the
[`reactions.get`](https://api.slack.com/methods/reactions.get) and
[`reactions.add`](https://api.slack.com/methods/reactions.add) Slack API
calls.

## Finding the matching `Rule` for an incoming `reaction_added` message

We need to iterate through our array of `Rule` objects to find the `Rule` that
matches the incoming message. Let's give this behavior its own method:

```js
Middleware.prototype.findMatchingRule = function(message) {
  var slackClient = this.slackClient;

  if (message && message.type === SlackClient.REACTION_ADDED &&
      message.item.type === 'message') {
    return this.rules.find(function(rule) {
      return rule.match(message, slackClient);
    });
  }
};
```

The first thing we do is assign `this.slackClient` to a new variable, since
[`this` will refer to a different object inside the
callback](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions#Lexical_this).

We wrap `this.rules.find` in a conditional to ensure that we call `Rule.match`
with a valid message. This could theoretically be part of the `Rule.match`
behavior itself. However, since the result of this check would remain the same
across all `Rule` objects for any message, it makes sense to implement it here.

## Testing `findMatchingRule`

Another benefit of writing this method first is that we can test its behavior
thoroughly and directly without calling `execute`. Since `execute` will be the
single function responsible for the entire application, it makes sense to
implement and test this step in isolation. [This will give us confidence that
all the corner cases are accounted for, without an exponential explosion in
the number of test
cases](http://googletesting.blogspot.com/2008/02/in-movie-amadeus-austrian-emperor.html).

Let's look at the first empty test case in
[`exercise/test/middleware-test.js`]({{ site.baseurl }}/exercise/test/middleware-test.js):

```js
describe('Middleware', function() {
  describe('findMatchingRule', function() {
    it('should find the rule matching the message', function() {
    });
  });
```

## Introducing the `sinon` test double library

The first thing we need is to instantiate a `Middleware` instance in our test
fixture. We'll also instantiate `Config` a config object, and we'll now
introduce the [`sinon` library](http://sinonjs.org/) to create a [test
double](http://googletesting.blogspot.com/2013/07/testing-on-toilet-know-your-test-doubles.html)
for `SlackClient`.

`sinon` is a library that can create stub, fake, and mock objects for you.
Though we wrote our own
[`SlackClientImplStub`]({{ site.baseurl }}/exercises/test/helpers/slack-client-impl-stub.js)
when testing the `Rule` class, we will need more objects and more behavior
when testing `Middleware`. As a result, we'll use `sinon` to create a double
for `SlackClient` in this test, rather than extracting `SlackClientImplStub`
into `test/helpers`.

Now add all of the necessary `require` statements, and instantiate the
`config`, `slackClient`, and `githubClient` objects:

```js
var Middleware = require('../lib/middleware');
var Config = require('../lib/config');
var GitHubClient = require('../lib/github-client');
var SlackClient = require('../lib/slack-client');
var helpers = require('./helpers');
var sinon = require('sinon');
var chai = require('chai');

describe('Middleware', function() {
  var config, slackClient, githubClient, middleware;

  beforeEach(function() {
    config = new Config(helpers.baseConfig());
    middleware = new Middleware(config, slackClient, githubClient);
  });
```

Notice that we leave the `githubClient` argument to the `Middleware`
constructor undefined for now. We'll define it when we add the behavior that
needs it.

## `reaction_added` test data

The second thing we need is a `reaction_added` message instance. Now that we're
familiar with the pattern of adding test data to our `test/helpers` package,
let's add the following to `exercise/test/helpers/index.js`:

```js
var SlackClient = require('../../lib/slack-client');

exports = module.exports = {
  REACTION: 'evergreen_tree',
  USER_ID: 'U5150OU812',

  // ...existing declarations...

  reactionAddedMessage: function() {
    return {
      type: SlackClient.REACTION_ADDED,
      user: exports.USER_ID,
      item: {
        type: 'message',
        channel: exports.CHANNEL_ID,
        ts: exports.TIMESTAMP
      },
      reaction: exports.REACTION,
      'event_ts': exports.TIMESTAMP
    };
  },

  // ...existing declarations...
```

Looking closely, we can see that there's also a new member to add to
`SlackClient`. Inside
[`exercise/lib/slack-client.js`]({{ site.baseurl }}/exercise/lib/slack-client.js)
add the following just below the constructor:

```js
// From: https://api.slack.com/events/reaction_added
// May get this directly from a future version of the slack-client package.
SlackClient.REACTION_ADDED = 'reaction_added';
```

This keeps with the theme of adding all Slack-related information and behavior
encapsulated within the `SlackClient` class. Of course, the most correct thing
would be to `require('slack-client')` and get the value that way. However, given
this is the only piece of information we need, we can [minimize
dependencies]({{ site.baseurl }}/concepts/minimizing-dependencies/) by
assigning this one constant value ourselves.

## The `findMatchingRule` test suite

With this helper data in place, we can now implement our first test:

```js
```

## Testing

## Check your work

By this point, all of the `Middleware` tests should be passing:

```sh
```

Now that you're all finished, compare your solutions to the code in
[`solutions/04-middleware/lib/github-client.js`]({{ site.baseurl }}/solutions/04-middleware/lib/github-client.js)
and
[`solutions/04-middleware/test/github-client-test.js`]({{ site.baseurl }}/solutions/04-middleware/test/github-client-test.js).

You may wish to `git commit` your work to your local repo at this point. After
doing so, try copying the `config.js` file from `solutions/04-middleware`
into `exercises` to see if it passes the test you wrote. Then run `git reset
--hard HEAD` and copy the test files instead to see if your implementation
passes.