# Puppeteer VS Unit-Testing your UI

In this Article I will show you another way of testing your UI Application with Puppeteer.
Even tough I personally use Angular the Article, examples and the message are independently from the concrete UI Framework.
It just pretends, that your UI can be run on Chrome/Chromium.

## The testing Pyramid

To get an Overview where Puppeteer might help you testing your angular App, lets take a quick look
at the well known testing Pyramid.

![Testing Pyramid](512px-Testing_Pyramid.svg.png)

The message usually is:
* Cover most parts of your Application with Unit-Tests
* Write lesser Integration-Tests
* Only if not already covered by the former tests, start writing some E2E/UI Tests

So what I dont like on the testing Pyramid shown above is:
1. They usually make no difference between UI and E2E Tests. But for me they are 2 different things.
2. There is much focus on writing Unit-Tests. But there are other ways to smarter test your application with less effort.

## The difference between E2E and UI Tests
E2E Tests mean to fully test your deployed environment including Frontend, Backend, Database. You can of course (and often have to) cheat a little bit with using mocks
especially to foreign services not part of your deployed environment.
For me it doesn't matter if you are testing changes in the database by directly accessing and querying the database or to check the correct values which were returned through the backend-services or displayed by the UI.
In contrast to the E2E Tests, UI-Test just test the UI. They can be run just against the UI Part of your application in abscence of any backend system.
For Angular users's fixture.nativeElement gives you the possibility to write UI-Tests. 

## The disadvantages of Angular Unit Testing
* The tests are due to their nature of being Unit Tests hardly aligned to the code. There is nothing wrong with this if you really have complex parts or functions in your project that have to be tested explicitly.
  But why not always write Unit-Tests?
* Unit-Test usually take some time to write. You have to mock many aspects of your Application like other components, services etc. your SUT uses. I see Unit-Tests which mock 90% of the code just to test the remaining 10%.
  Keeping this mock infrastructure always in sync with your code over the time of your projects requires a significant amount of time and effort. For most developers i encountered this usually not the likelies part to do.
* As the result of this hard alignment to the code refactoring becomes cumberstone. Be honest, most tests fail because of refactoring. You have to rewrite them, change them, adjust the Mocks you are using etc. This leads to following drawback.
* Who tells you after code refactorings that the application works as before when you to modify big parts of the tests? What use they give to you?

Unit Tests of course have their advantages too - you can read them on other pages about Unit-Tests. One main advantage is their robustness, namely they always show the same
results independly form where the are run (usually locally or on a CI platform). The second one it their execution speed.

I totally agree with this. But I think there is a ways to combine the advantages without having much of the disadvantages shown above in using UI-Tests.

## Puppeteer to the rescue

> Puppeteer is a Node library which provides a high-level API to control Chromium or Chrome over the DevTools Protocol.

Taken from https://pptr.dev.

Puppeteer is often seen as an equivalent of Selenium, which ist more stable, faster, has more control over the Browser but relies on Chrome.
But especially the Point "Control the Browser" makes Puppeteer useable in Scenarios not reachable by Selenium: The UI-Tests.

The missing piece in the puzzle comes with puppeteers ability to mock Browser requests and respond with fake data.
This helps run your and test your UI independently from any backend.

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

You are now able to run your Application in abscense of any Backend System and can control the responses from the requests your UI ist sending.
This enables to to completly UI Test your Application with Pupperteer. I used Jest as my Testing framework.

## Comparing Puppeteers UI-Tests with Unit-Tests
* UI-Tests are written on a highler level than Unit-Tests. This makes them more robust to refactorings. You could just swap your current UI-Framework from
  say Angular to Vue or Flex and the Tests will still run, pretending the selectors you are using are kept stable.
* As such you can rely on your Test suite after finished your code refactorings. If all tests run green you know that the application behaves as before.
* You dont need to write and maintain a mock infratructure. In most cases it is sufficient to just mock your browsers XHR-Request to the backend services. But with puppeteer you can even control more parts of your browser if that isnt enough.
* UI Tests written in Puppeteer are usually very stable and fast, compareable to Unit Tests.


### Run Puppeteer on a CI-Server (like Jenkins)

To actually run a Jest/Puppeteer session on your favorite UI Server you have a couple of opportunities.
For me it made sense to first build the project to just have all static content in one folder. In case of Angular `npm run build` or `ng serve` will do the trick leaving all the transpiled source files and other files in the `/dist` folder.

Having then all your required files ready to deliver in the `dist` folder you can setup a webserver of your choice
to serve the static contents for Jest/Puppeteer to run their tests on top of it.

The webserver I use is express. The hard part here is to actually stop express but there are some libraries to help you there. [Stoppable](https://www.npmjs.com/package/stoppable) uses a wrapper around the HttpServer instance to implement the `close()` function.

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


## Conclusion and Suggestions
Angular testing is great and I was a big fan of it for a long time. But it has its drawbacks and I encounter often that
Projects are reeling on a big mock infrastructure to isolate their components and make them Unit-testable. The mocks by itself must be maintained and kept in sync with the current implementation.
This makes the code harder to refactor and you cannot rely on your test after refactoring the code. They are just testing the current working snapshot of your code.

Much of the testing afford can now be done with UI-Testing using Puppeteer. You test your code at a higher level being more robust to refactorings.
Of course you loose the ability to explicitly white box test functions. But there is no one permitting you from writing same.

So, my suggention is to be pragmatic. Carefully adjust your testing efford to the following aspects:
  * How big is the team actually wrtiting the UI-Part of your application
  * How much impact and number of users the application have
  * How fast will you develope your app and hit the marked? Do you need a near 100% Unit Test Coverage if you are usure whether your application will survive (MVP approach)?
  * What is the TTL of your application in todays fast changing world
  * How much is the Code refactored / stability of the code base (-> More UI-Tests, less Unit-Tests)

As such i dont like to generalise the testing pyramid on each project and for UI Tests it has to be overthought in my opinion.
