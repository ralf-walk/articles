# Puppeteer VS unit testing your UI

![Puppet Man](puppet_man_human_fighter_robin_hood_doll_puppeteer_acting-1350656.jpg)
https://pxhere.com/en/photo/1350656

In this Article I will show you another way of testing your UI Application with Puppeteer.

Although I personally use Angular in this article, all the Puppeteer examples and the message are valid for other UI frameworks as well.
It just pretends, that your UI can be run on Chrome/Chromium.

## The testing Pyramid

To get an Overview where Puppeteer might help you testing your UI application, lets take a quick look
at the well known testing Pyramid.

![Testing Pyramid](512px-Testing_Pyramid.svg.png)

The message usually is:
* Cover most parts of your application with unit tests
* Write lesser Integration-Tests
* Only if not already covered by the former tests, start writing some E2E/UI Tests

I think the disadvantages of the testing Pyramid shown above are:
1. They usually make no difference between UI and E2E tests. But for me that are two different things.
2. There is too much focus on writing unit tests. But there are other ways to smarter test your application with less effort.

## The difference between E2E and UI Tests
E2E tests mean to fully test your deployed environment including frontend, backend and database. You can of course (and often have to) cheat a little bit with using mocks especially to foreign services not part of your deployed environment.
For me it doesn't matter if you are testing changes in the database by directly accessing and querying the database or by checking the correct values which were returned through the backend services or displayed by the UI.
In contrast to the E2E tests, UI test just test the UI. They can be run just against the UI part of your application in absence of any backend system.

## The disadvantages of unit testing

For Angular [Angular Unit Testing](https://angular.io/guide/testing) is the most common way to unit test your application. Usually you have a separate test file for each angular component / service. Other UI frameworks follow similar approaches. The benefits are:

* All tests are encapsulated with their respective components / services. This level of isolation makes the tests easy to manage. You don't have any side effects from other parts of your application and can focus on just a small part.
* The tests are robust, namely they always show the same results independently from where they run (usually locally or on a CI platform).
* The execution speed is usually very high. You get fast feedback and they are often executed before checking in the code.

So the tests are due to their nature of being unit tests hardly aligned to the code. There is nothing wrong with this if you really have complex parts or functions in your project that have to be tested explicitly.
But why not always write unit tests?

* Unit tests usually take some time to write. You have to mock many aspects of your application like other components, services etc. your SUT uses. I see unit tests which mock 90% of the code just to test the remaining 10%.
  Keeping this mock infrastructure always in sync with your code over the time of your projects requires a significant amount of time and effort. For most developers I encountered this usually is the most unpreferred part to do.
* As result of this hard alignment to the code refactoring becomes cumberstone. Be honest, most tests fail because of refactoring existing code or developing new features (forgot the mock huh?). You have to rewrite them, change them, adjust the mocks you are using etc. This leads to the following drawback.
* Who tells you after code refactorings that the application works as before when you to modify big parts of the tests? What value they give to you?

I totally agree with this. But I think there is a way to combine the advantages without having too much of the disadvantages shown above in using UI tests.

## Puppeteer to the rescue

> Puppeteer is a Node library which provides a high-level API to control Chromium or Chrome over the DevTools Protocol.

https://pptr.dev.

Puppeteer is often seen as an equivalent of Selenium, which is more stable, faster, has more control over the browser but relies on Chrome.
But especially the Point "Control the Browser" makes Puppeteer useable in scenarios not reachable by Selenium: The UI tests.

The missing piece in the puzzle comes with puppeteers ability to mock browser requests and respond with fake data.
This helps run your UI tests independently from any backend.

To mock requests you do simply:

```javascript
await page.setRequestInterception(true);
page.on('request', async request => {

  if (request.url().endsWith("/api/backendcall")) {
    await request.respond({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ key: 'value' })
    });
  } else {
    await request.continue();
  }
});
```

You are now able to run your application in abscense of any backend system and can control the responses from the requests your UI ist sending.
This enables to completly UI test your application with Pupperteer. I used [Jest](https://jestjs.io/) as my testing framework.

## Comparing Puppeteers UI-Tests with unit tests
* UI tests are written on a highler level than unit tests. This makes them more robust to refactorings. You could just swap your current UI framework from e.g. Angular to Vue or Flex and the tests will still run, pretending the selectors you are using are kept stable.
* Therefore you can rely on your test suite after finished your code refactorings. If all tests run green you know that the application behaves as before.
* You don't need to write and maintain a mock infrastructure. In most cases it is sufficient to just mock your browsers XHR-Request to the backend services. But with Puppeteer you can even control more parts of your browser if that isn't enough.
* UI tests written in Puppeteer are usually very stable and fast, compareable to unit tests.


### Run Puppeteer on a CI-Server (like Jenkins)

To actually run a Jest/Puppeteer session on your favorite UI server you have a couple of opportunities.
For me it made sense to first build the project to just have all static content in one folder. In case of Angular `npm run build` or `ng serve` will do the trick leaving all the transpiled source files and other files in the `/dist` folder.

Having all your required files ready to deliver in the `dist` folder you can setup a webserver of your choice
to serve the static contents for Jest/Puppeteer to run their tests on top of it.

The webserver I use is express. The difficulty is to actually stop express but there are some libraries to help you. [Stoppable](https://www.npmjs.com/package/stoppable) uses a wrapper around the HttpServer instance to implement the `close()` function.

Installation:

```
npm i express stoppable --save-dev
```

```javascript
const express = require('express');
const stoppable = require('stoppable');

async function createServer(port) {
  const app = express();
  app.use(express.static('dist'));
  return new Promise((resolve, reject) => {
    const server = app.listen(port, () => resolve(stoppable(server)));
  });
}
```

Somewhere in your Jest Callback you could have:

```javascript
let server;

beforeAll(async () => {
  server = await createServer(4200);
});

afterAll(async () => {
  await server.close();
});
```


## Conclusion and suggestions
Angular testing is great and I was a big fan of it for a long time. But it has its drawbacks and I encounter often that
projects are relying on a big mock infrastructure to isolate their components and make them unit testable. The mocks by itself must be maintained and kept in sync with the current implementation.
This makes the code more difficult to refactor and you cannot rely on your test after refactoring the code. They are just testing the current working snapshot of your code.

Much of the testing afford can now be done with UI testing using Puppeteer. You test your code at a higher level being more robust to refactorings.
Of course you loose the ability to explicitly white box test functions. But there is no one permitting you from writing them.

So, my suggestion is to be pragmatic. Carefully adjust your testing efford to the following aspects:
  * How big is the team actually writing the UI part of your application?
  * How much impact and number of users the application have?
  * How fast will you develope your app and hit the marked? Do you need a near 100% unit test coverage if you are not sure whether your application will survive (MVP approach)?
  * What is the TTL of your application in today's fast changing world?
  * How much is the code refactored / stability of the code base (-> More UI-Tests, less unit tests)?

As such I don't like to generalise the Test Pyramid on each project.
