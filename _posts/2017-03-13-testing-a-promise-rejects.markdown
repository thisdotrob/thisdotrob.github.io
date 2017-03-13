---
layout: post
title:  "Testing that a promise rejects in Javascript"
date:   2017-03-13 20:38:00 +0100
categories: unit testing mocha promises
---

I came across a typical unit test case today which I have covered many a time before: testing that a promise returning function rejects under a certain scenario.

For the sake of an example, lets say this promise returning function is going to be responsible for hitting an API and returning the result. I would normally write the following test (using [mocha](https://mochajs.org/)):

```javascript
describe('when the API returns a 500 status code', () => {
  it('should reject with an error', () => {
    nock('http://mydomain.com')
      .get('/api/resource')
      .reply(500, 'This API is borked');

    return getResult()
      .then(() => throw new Error('Promise did not reject :o'))
      .catch((err) => assert.equal(err.message, '500 - "This API is borked"'));
  });
});
```

By adding the ```should.fail()``` in the ```then``` block we make sure that if the promise resolves instead of rejects, the test will fail as it should. Lets write the function that should make this test fail:

```javascript
function getResult() {
  return new Promise((resolve) => {
    request('http://mydomain.com/api/resource', (error, response, body) => {
      resolve(body);
    });
  });
}
```

Here we simply use the [request](https://www.npmjs.com/package/request) module to make a request to our API, and resolve with the body (if it exists). Note we aren't rejecting if the error exists in the callback, so our unit test should fail.

Sure enough the test fails, but the failure message is a bit confusing:

```bash
AssertionError: expected 'Promise did not reject :o' to be '500 - "This API is borked"'
```

The error we throw in the test if the promise resolves is being caught by the catch block, and making the assertion on the error's message fail.

I discovered we can avoid this if we rewrite the test to give the ```then``` method a second ```onRejected``` callback instead of chaining a catch onto it:

```javascript
describe('when the API returns a 500 status code', () => {
  it('should reject with an error', () => {
    nock('http://mydomain.com')
      .get('/api/resource')
      .reply(500, 'This API is borked');

    return getResult()
      .then(() => {
        throw new Error('Promise did not reject :o');
      }, (err) => {
        assert.equal(err.message, '500 - "This API is borked"');
      });
  });
});
```

The error is no longer caught and then run through the assertion, and we get a clearer message from the test runner:

```bash
Error: Promise did not reject :o
```

We then discovered an even cleaner way to write the test, taking advantage of [Should's](https://shouldjs.github.io/) promise specific assertions:

```javascript
describe('when the API returns a 500 status code', () => {
  it('should reject with an error', () => {
    nock('http://mydomain.com')
      .get('/api/resource')
      .reply(500, 'This API is borked');

    return getResult().should.be.rejectedWith('500 - "This API is borked"')
  });
});
```

The output is still clear and informative, but the test is much more readable:

```bash
AssertionError: expected [Promise] to be rejected
```

If you don't want to use an assertion library such as should, its best to stick to the test example that uses the double callback version of the ```then``` method. However in my opinion it is well worth using a library for this exact reason, as the tests are much more readable.
