# Cypress Guide

This is the setup I've landed on after a few years building and rebuilding Cypress frameworks across different projects. Some of it I learned from docs, most of it I learned from breaking things in CI at 11pm. I'm writing it the way I'd explain it to someone joining the team who already knows how to test but hasn't touched Cypress.

There's no fluff here. If I've kept something in, it's because I actually use it.

## Table of Contents

1. [Why Cypress (and Where It Doesn't Fit)](#1-why-cypress-and-where-it-doesnt-fit)
2. [Prerequisites and Installation](#2-prerequisites-and-installation)
3. [Project Structure](#3-project-structure)
4. [Cypress Configuration](#4-cypress-configuration)
5. [Locator Strategy](#5-locator-strategy)
6. [Actions and Assertions](#6-actions-and-assertions)
7. [Parent-Child Command Chains](#7-parent-child-command-chains)
8. [Hooks](#8-hooks)
9. [Page Objects vs App Actions](#9-page-objects-vs-app-actions)
10. [Custom Commands](#10-custom-commands)
11. [Fixtures: Static Test Data](#11-fixtures-static-test-data)
12. [Data Factories: Dynamic Test Data](#12-data-factories-dynamic-test-data)
13. [Utilities and Helpers](#13-utilities-and-helpers)
14. [Environment Configuration](#14-environment-configuration)
15. [API Testing with cy.request and cy.intercept](#15-api-testing-with-cyrequest-and-cyintercept)
16. [Authentication Patterns](#16-authentication-patterns)
17. [Tagging Strategy](#17-tagging-strategy)
18. [Mochawesome Reporting](#18-mochawesome-reporting)
19. [CI/CD Integration](#19-cicd-integration)
20. [Parallelization](#20-parallelization)
21. [Debugging](#21-debugging)
22. [Best Practices](#22-best-practices)
23. [Code Review Checklist](#23-code-review-checklist)
24. [Maintenance](#24-maintenance)
25. [Useful Plugins](#25-useful-plugins)
26. [Closing Thoughts](#26-closing-thoughts)

## 1. Why Cypress (and Where It Doesn't Fit)

If your app lives mostly on one domain and your team writes JavaScript, it's a really good fit. The developer experience is genuinely good: time-travel debugging, automatic retries, a visual test runner that actually makes debugging enjoyable. Developers will write tests in it because it doesn't feel like a chore.

Where it gets awkward: multi-domain flows. Cypress technically supports multiple origins now but it's fiddly, and Playwright handles it more naturally. If Safari coverage is critical, Playwright has better Safari support. And for heavy parallelization without paying for Cypress Cloud, Playwright shards natively while Cypress needs either Cypress Cloud or a third-party plugin.

I use Cypress for SPAs and Playwright for complex enterprise flows that cross domains. This guide assumes Cypress fits your situation.

## 2. Prerequisites and Installation

You'll need Node.js 18 or higher (I use 20 LTS), npm 9+, and an editor. VS Code with ESLint is what I use.

```bash
node -v
npm -v
```

Installation:

```bash
mkdir my-cypress-framework
cd my-cypress-framework
npm init -y
npm install cypress --save-dev
npx cypress open
```

When the wizard launches, pick E2E Testing, let Cypress scaffold the folder structure, pick Chrome, and skip the example specs. You don't need them and they'll clutter your `e2e/` folder.

Here are the `package.json` scripts I add to every project from the start:

```json
{
  "scripts": {
    "cy:open": "cypress open",
    "cy:run": "cypress run",
    "cy:smoke": "cypress run --env grepTags=@smoke",
    "cy:regression": "cypress run --env grepTags=@regression",
    "cy:staging": "cypress run --env configFile=staging",
    "cy:prod": "cypress run --env configFile=prod",
    "pretest": "rimraf cypress/reports",
    "test": "npm run cy:run",
    "posttest": "npm run report:merge && npm run report:generate",
    "report:merge": "mochawesome-merge cypress/reports/mocha/*.json > cypress/reports/output.json",
    "report:generate": "marge cypress/reports/output.json -f report -o cypress/reports/html --inline"
  }
}
```

And the `.gitignore` additions. Do this before your first commit or you'll spend time undoing it later:

```
node_modules/
cypress/videos/
cypress/screenshots/
cypress/reports/
cypress/downloads/
.env
.env.*
*.log
```

Don't commit videos, screenshots, or reports. They balloon the repo size and add nothing to source control. If someone needs to see a failure, it should come from CI artifacts.

## 3. Project Structure

This is the structure I've settled on. It's grown from small projects and held up at scale:

```
my-cypress-framework/
├── cypress/
│   ├── e2e/
│   │   ├── auth/
│   │   │   ├── login.cy.js
│   │   │   └── signup.cy.js
│   │   ├── checkout/
│   │   │   ├── cart.cy.js
│   │   │   └── payment.cy.js
│   │   └── admin/
│   │       └── user-management.cy.js
│   │
│   ├── fixtures/
│   │   ├── users.json
│   │   └── api-mocks/
│   │       └── orders-success.json
│   │
│   ├── support/
│   │   ├── e2e.js
│   │   ├── commands.js
│   │   ├── pages/
│   │   │   ├── LoginPage.js
│   │   │   └── CheckoutPage.js
│   │   ├── factories/
│   │   │   ├── userFactory.js
│   │   │   └── productFactory.js
│   │   └── utils/
│   │       ├── dateHelper.js
│   │       └── apiClient.js
│   │
│   ├── config/
│   │   ├── dev.json
│   │   ├── staging.json
│   │   └── prod.json
│   │
│   └── reports/
│
├── .github/workflows/
│   └── cypress.yml
│
├── cypress.config.js
├── package.json
└── README.md
```

A few things I feel strongly about here. Group tests by feature, not by test type. Folders should reflect the app auth, checkout, admin and not "smoke" or "regression". Those belong in tags, not folders. One spec per behavior. If a spec file is getting close to 400 lines, it's probably doing too much.

`support/pages` holds Page Objects. `support/factories` is for data builders using Faker. `support/utils` is for pure JS helpers that have no Cypress dependency whatsoever. `config/` holds environment-specific JSON that `cypress.config.js` loads at runtime.

## 4. Cypress Configuration

```javascript
const { defineConfig } = require('cypress');
const fs = require('fs-extra');
const path = require('path');

function getConfigByEnv(env) {
  const file = path.resolve(`cypress/config/${env}.json`);
  if (!fs.existsSync(file)) {
    throw new Error(`Config file not found: ${file}`);
  }
  return fs.readJsonSync(file);
}

module.exports = defineConfig({
  reporter: 'cypress-mochawesome-reporter',
  reporterOptions: {
    charts: true,
    reportPageTitle: 'Cypress E2E Report',
    embeddedScreenshots: true,
    inlineAssets: true,
    reportDir: 'cypress/reports/mocha',
  },

  viewportWidth: 1440,
  viewportHeight: 900,
  defaultCommandTimeout: 8000,
  requestTimeout: 10000,
  responseTimeout: 30000,
  pageLoadTimeout: 60000,
  watchForFileChanges: false,
  retries: {
    runMode: 2,
    openMode: 0,
  },
  video: true,
  screenshotOnRunFailure: true,
  trashAssetsBeforeRuns: true,

  e2e: {
    baseUrl: 'https://staging.myapp.com',
    specPattern: 'cypress/e2e/**/*.cy.js',
    supportFile: 'cypress/support/e2e.js',

    setupNodeEvents(on, config) {
      require('cypress-mochawesome-reporter/plugin')(on);
      require('@cypress/grep/src/plugin')(config);

      const envName = config.env.configFile || 'staging';
      const envConfig = getConfigByEnv(envName);
      config.baseUrl = envConfig.baseUrl;
      config.env = { ...config.env, ...envConfig };

      on('task', {
        log(message) {
          console.log(message);
          return null;
        },
      });

      return config;
    },
  },
});
```

A few things worth flagging. Don't hardcode `baseUrl` in spec files that's what the config loader is for. Don't set `defaultCommandTimeout` to 60 seconds. I've seen that "fix" hide intermittent timing issues for months. And don't set retries higher than 2. If a test needs 3 retries to pass, it's not a test, it's a coin flip.

## 5. Locator Strategy

If there's one thing to get right early, it's this. Locators are responsible for the majority of maintenance headaches on every project I've worked on.

```javascript
// Best option dedicated test attribute
cy.get('[data-cy="submit-btn"]')

// Acceptable
cy.contains('button', 'Submit')
cy.get('input[type="email"]')

// Use sparingly
cy.get('.submit-button')

// Avoid
cy.get('div > div > div > span:nth-child(3)')
cy.xpath('//div[@class="container"]/div[2]')
```

`data-cy` attributes win because they're never used for styling and never used for analytics, so they survive refactors. They also signal intent: this element exists, in part, so tests can find it. When I've worked on teams that skip this, the locator files turn into a graveyard of class names that changed two sprints ago.

The naming convention I use:

```
[component]-[element]-[action]

data-cy="login-form-email-input"
data-cy="login-form-submit-button"
data-cy="header-logout-link"
data-cy="cart-item-remove-button"
```

Get devs to add these to interactive elements and make it part of code review. It's a low-effort habit with a disproportionate effect on how long the suite stays stable.

Add this custom command and use it everywhere:

```javascript
// cypress/support/commands.js
Cypress.Commands.add('getByCy', (selector, ...args) => {
  return cy.get(`[data-cy="${selector}"]`, ...args);
});

// Usage
cy.getByCy('login-submit').click();
```

## 6. Actions and Assertions

### Actions

```javascript
// Navigation
cy.visit('/login');
cy.go('back');
cy.reload();

// Typing
cy.getByCy('email-input').type('user@test.com');
cy.getByCy('email-input').clear();
cy.getByCy('search').type('laptop{enter}');
cy.getByCy('password').type('secret', { log: false });   // hide in logs

// Clicking
cy.getByCy('submit').click();
cy.getByCy('submit').click({ force: true });   // last resort see note below
cy.getByCy('image').dblclick();

// Checkboxes and selects
cy.getByCy('terms').check();
cy.getByCy('newsletter').uncheck();
cy.getByCy('country').select('Ireland');

// File upload
cy.getByCy('upload').selectFile('cypress/fixtures/photo.png');

// Hover and focus
cy.getByCy('menu').trigger('mouseover');
cy.getByCy('search').focus();

// Scrolling
cy.getByCy('footer').scrollIntoView();
cy.scrollTo('bottom');
```

About `{ force: true }`: it bypasses Cypress's safety checks. Every time I've reached for it, it's either been a real UX bug (element not actually reachable) or a timing issue I needed to fix properly. Leave a comment explaining why whenever you use it.

### Assertions

```javascript
// Existence and visibility
cy.getByCy('welcome').should('exist');
cy.getByCy('loader').should('not.be.visible');

// Text
cy.getByCy('welcome').should('have.text', 'Welcome, Alice');
cy.getByCy('welcome').should('contain.text', 'Welcome');

// Attributes and values
cy.getByCy('email-input').should('have.value', 'user@test.com');
cy.getByCy('submit-btn').should('be.disabled');
cy.getByCy('terms').should('be.checked');

// Count
cy.getByCy('product-card').should('have.length', 5);

// URL
cy.url().should('include', '/dashboard');

// Chained
cy.getByCy('cart-badge')
  .should('be.visible')
  .and('have.text', '3');
```

One thing that trips people up coming from Selenium: you don't write explicit waits. Every assertion retries automatically until it passes or the timeout is hit. If you feel the urge to write `cy.wait(3000)`, that's Cypress telling you to assert on the condition you're actually waiting for instead.

## 7. Parent-Child Command Chains

This is the mental model shift that makes Cypress click. It tripped me up for a while.

Parent commands start a fresh chain:

```javascript
cy.visit('/login');
cy.get('button');
cy.url();
cy.request('/api/users');
```

Child commands operate on whatever the previous command yielded:

```javascript
cy.get('form')
  .find('input')
  .first()
  .type('hello')
  .should('have.value', 'hello');
```

When you need the actual value from an element or response, use `.then()`:

```javascript
cy.getByCy('email-input').then(($input) => {
  const value = $input.val();
  expect(value).to.equal('user@test.com');
});

cy.request('/api/user').then((response) => {
  expect(response.status).to.eq(200);
  expect(response.body.email).to.eq('user@test.com');
});
```

For values you'll reuse across a test, alias them with `.as()`:

```javascript
beforeEach(() => {
  cy.fixture('users').as('users');
  cy.intercept('GET', '/api/products').as('getProducts');
});

it('logs in standard user', function () {
  // Use function() not arrow needed to access `this`
  const user = this.users.standard;
  cy.visit('/login');
  cy.getByCy('email').type(user.email);
  cy.wait('@getProducts');
});
```

The async trap that catches everyone at least once:

```javascript
// Wrong Cypress commands are queued, they don't return values
const text = cy.getByCy('greeting').text();
console.log(text);   // undefined

// Right
cy.getByCy('greeting').then(($el) => {
  console.log($el.text());
});

// Usually better.. just assert
cy.getByCy('greeting').should('have.text', 'Hello, Alice');
```

## 8. Hooks

```javascript
describe('Login flow', () => {

  before(() => {
    cy.task('db:seed');
  });

  beforeEach(() => {
    cy.visit('/login');
  });

  afterEach(() => {
    // cleanup per test if needed
  });

  after(() => {
    cy.task('db:cleanup');
  });

  it('valid login redirects to dashboard', () => { /* ... */ });
});
```

Execution order:

```
before
  beforeEach → test 1 → afterEach
  beforeEach → test 2 → afterEach
after
```

I default to `beforeEach` over `before`. `before` shares state across tests, and shared state causes order dependencies, which eventually causes flakiness. The only time I reach for `before` is for genuinely expensive one-time setup like seeding a database with a large dataset.

## 9. Page Objects vs App Actions

This debate gets weirdly heated. Here's where I've landed.

Page Objects work well for UI flows you actually want to verify end-to-end. The login test should go through the UI. The checkout flow test should go through the UI. But your dashboard test shouldn't go through the login UI. If login is broken, the login test should fail not every test in the suite.

### Page Object Model

```javascript
// cypress/support/pages/LoginPage.js
class LoginPage {
  emailInput()    { return cy.getByCy('email-input'); }
  passwordInput() { return cy.getByCy('password-input'); }
  submitButton()  { return cy.getByCy('login-submit'); }
  errorMessage()  { return cy.getByCy('login-error'); }

  visit() {
    cy.visit('/login');
    return this;
  }

  fillEmail(email) {
    this.emailInput().clear().type(email);
    return this;
  }

  fillPassword(password) {
    this.passwordInput().clear().type(password, { log: false });
    return this;
  }

  submit() {
    this.submitButton().click();
    return this;
  }

  login(email, password) {
    return this.fillEmail(email).fillPassword(password).submit();
  }
}

export default new LoginPage();
```

```javascript
// cypress/e2e/auth/login.cy.js
import loginPage from '../../support/pages/LoginPage';

describe('Login flow', () => {
  it('valid credentials redirect to dashboard', () => {
    loginPage.visit().login('user@test.com', 'password123');
    cy.url().should('include', '/dashboard');
  });
});
```

Note: locators as methods, not properties. If you define them as properties (e.g., `this.emailInput = cy.get(...)`), they get evaluated at class instantiation time and can cause issues with Cypress's retry mechanism.

### App Actions

For setup that isn't part of what you're testing, bypass the UI entirely:

```javascript
// cypress/support/commands.js
Cypress.Commands.add('loginByApi', (email, password) => {
  cy.request('POST', '/api/auth/login', { email, password }).then((response) => {
    window.localStorage.setItem('authToken', response.body.token);
  });
});

// In tests
describe('Dashboard', () => {
  beforeEach(() => {
    cy.loginByApi('user@test.com', 'password123');
    cy.visit('/dashboard');
  });

  it('shows user name', () => {
    cy.getByCy('user-name').should('contain', 'Alice');
  });
});
```

Use both. Page Objects for flows you're actually testing. App Actions for setup.

## 10. Custom Commands

Custom commands extend the `cy` API. I use them for cross-cutting concerns, repeated locator patterns, and anything that involves Cypress and needs to stay out of Page Objects.

```javascript
// cypress/support/commands.js

Cypress.Commands.add('getByCy', (selector) => {
  return cy.get(`[data-cy="${selector}"]`);
});

Cypress.Commands.add('loginByApi', (email, password) => {
  cy.session([email, password], () => {
    cy.request({
      method: 'POST',
      url: `${Cypress.env('apiUrl')}/auth/login`,
      body: { email, password },
    }).then(({ body }) => {
      window.localStorage.setItem('authToken', body.token);
    });
  });
});

Cypress.Commands.add('waitForApp', () => {
  cy.getByCy('app-loaded').should('exist');
});
```

When deciding between a custom command and a utility function: if it involves `cy.*`, make it a command. If it's pure logic data transformation, string formatting, date math make it a utility function.

```javascript
// utility no Cypress
export const formatCurrency = (cents) => `$${(cents / 100).toFixed(2)}`;

// custom command uses Cypress
Cypress.Commands.add('addToCart', (productId) => {
  cy.request('POST', '/api/cart', { productId });
});
```

## 11. Fixtures: Static Test Data

Fixtures live in `cypress/fixtures/` as JSON files.

```json
// cypress/fixtures/users.json
{
  "standard": {
    "email": "user@test.com",
    "password": "Pass123!",
    "name": "Alice"
  },
  "admin": {
    "email": "admin@test.com",
    "password": "Admin123!",
    "name": "Bob"
  }
}
```

```javascript
describe('Login', () => {
  beforeEach(() => {
    cy.fixture('users').as('users');
  });

  it('standard user logs in', function () {
    cy.visit('/login');
    cy.getByCy('email').type(this.users.standard.email);
  });
});
```

Fixtures are also the cleanest way to mock API responses:

```javascript
it('shows products from API', () => {
  cy.intercept('GET', '/api/products', { fixture: 'api-mocks/products-list.json' }).as('getProducts');
  cy.visit('/shop');
  cy.wait('@getProducts');
});
```

Fixtures are good for: API response mocks, static reference data, stable test users. They're not good for: dynamic data with UUIDs or timestamps, data that changes per environment, credentials (use env vars for those).

## 12. Data Factories: Dynamic Test Data

This is where most teams stop short. Fixtures work for stable data but fall apart in parallel runs you get collisions, or tests that pass alone but fight each other in CI. Factories solve this by generating fresh, unique data on demand.

```bash
npm install --save-dev @faker-js/faker
```

```javascript
// cypress/support/factories/userFactory.js
import { faker } from '@faker-js/faker';

const buildUser = (overrides = {}) => ({
  id: faker.string.uuid(),
  firstName: faker.person.firstName(),
  lastName: faker.person.lastName(),
  email: faker.internet.email().toLowerCase(),
  password: 'Test@1234',
  phone: faker.phone.number(),
  address: {
    street: faker.location.streetAddress(),
    city: faker.location.city(),
    country: 'Ireland',
    zip: faker.location.zipCode(),
  },
  role: 'user',
  isActive: true,
  ...overrides,
});

const buildAdmin = (overrides = {}) => buildUser({ role: 'admin', ...overrides });

const buildList = (count, overrides = {}) =>
  Array.from({ length: count }, () => buildUser(overrides));

export const userFactory = {
  build: buildUser,
  buildAdmin,
  buildList,
};
```

```javascript
// cypress/support/factories/productFactory.js
import { faker } from '@faker-js/faker';

const CATEGORIES = ['electronics', 'clothing', 'books', 'home'];

const buildProduct = (overrides = {}) => ({
  id: faker.string.uuid(),
  sku: faker.string.alphanumeric(8).toUpperCase(),
  name: faker.commerce.productName(),
  price: parseFloat(faker.commerce.price({ min: 10, max: 500 })),
  category: faker.helpers.arrayElement(CATEGORIES),
  inStock: faker.datatype.boolean({ probability: 0.8 }),
  stockCount: faker.number.int({ min: 0, max: 100 }),
  ...overrides,
});

export const productFactory = {
  build: buildProduct,
  buildOutOfStock: (overrides) => buildProduct({ inStock: false, stockCount: 0, ...overrides }),
  buildList: (count, overrides) => Array.from({ length: count }, () => buildProduct(overrides)),
};
```

Using them in tests:

```javascript
import { userFactory } from '../../support/factories/userFactory';
import { productFactory } from '../../support/factories/productFactory';

describe('Signup', () => {
  it('new user can sign up', () => {
    const newUser = userFactory.build();
    cy.visit('/signup');
    cy.getByCy('first-name').type(newUser.firstName);
    cy.getByCy('email').type(newUser.email);
    cy.getByCy('password').type(newUser.password);
    cy.getByCy('submit').click();
  });

  it('renders 20 products', () => {
    const products = productFactory.buildList(20);
    cy.intercept('GET', '/api/products', { body: products }).as('getProducts');
    cy.visit('/shop');
    cy.wait('@getProducts');
    cy.getByCy('product-card').should('have.length', 20);
  });
});
```

A few things I always do with factories: always allow overrides through spread (`{ ...defaults, ...overrides }`), always generate IDs as UUIDs, and never randomize the password use a fixed test password or you'll spend time debugging auth failures caused by a password that happened to fail validation. Keep factories as pure JS, no Cypress dependencies.

## 13. Utilities and Helpers

Pure functions, no Cypress. I put anything here that I'd be able to unit test without a browser.

```javascript
// cypress/support/utils/dateHelper.js
export const formatDate = (date, format = 'YYYY-MM-DD') => {
  const d = new Date(date);
  const year = d.getFullYear();
  const month = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return format.replace('YYYY', year).replace('MM', month).replace('DD', day);
};

export const addDays = (date, days) => {
  const d = new Date(date);
  d.setDate(d.getDate() + days);
  return d;
};
```

```javascript
// cypress/support/utils/stringHelper.js
export const generateUniqueEmail = (prefix = 'test') =>
  `${prefix}+${Date.now()}@example.com`;

export const slugify = (str) =>
  str.toLowerCase().replace(/[^\w\s-]/g, '').replace(/\s+/g, '-');
```

```javascript
// cypress/support/utils/apiClient.js
const apiRequest = (method, endpoint, body = null) => {
  return cy.request({
    method,
    url: `${Cypress.env('apiUrl')}${endpoint}`,
    body,
    headers: {
      Authorization: `Bearer ${window.localStorage.getItem('authToken')}`,
    },
    failOnStatusCode: false,
  });
};

export const api = {
  get: (endpoint) => apiRequest('GET', endpoint),
  post: (endpoint, body) => apiRequest('POST', endpoint, body),
  put: (endpoint, body) => apiRequest('PUT', endpoint, body),
  delete: (endpoint) => apiRequest('DELETE', endpoint),
};
```

My rule of thumb: if I copy the same helper logic three times in tests, it goes in `utils/`.

## 14. Environment Configuration

```json
// cypress/config/dev.json
{
  "baseUrl": "https://dev.myapp.com",
  "apiUrl": "https://dev-api.myapp.com",
  "users": {
    "standard": "qa-dev@company.com"
  }
}
```

```json
// cypress/config/staging.json
{
  "baseUrl": "https://staging.myapp.com",
  "apiUrl": "https://staging-api.myapp.com",
  "users": {
    "standard": "qa-staging@company.com"
  }
}
```

The loader is in `cypress.config.js` (see section 4). To run against a specific environment:

```bash
npm run cy:run -- --env configFile=staging
npm run cy:run -- --env configFile=dev
```

For sensitive values, never commit them. Use environment variables:

```bash
export CYPRESS_TEST_USER_PASSWORD='secretValue'
npx cypress run
```

```javascript
cy.getByCy('password').type(Cypress.env('TEST_USER_PASSWORD'), { log: false });
```

## 15. API Testing with cy.request and cy.intercept

### cy.request

Direct HTTP calls are good for API-only tests, seeding data, or verifying backend state.

```javascript
describe('User API', () => {
  it('GET /users returns list', () => {
    cy.request('GET', '/api/users').then((response) => {
      expect(response.status).to.eq(200);
      expect(response.body).to.be.an('array');
    });
  });

  it('POST /users creates user', () => {
    const newUser = userFactory.build();
    cy.request('POST', '/api/users', newUser).then((response) => {
      expect(response.status).to.eq(201);
      expect(response.body.email).to.eq(newUser.email);
    });
  });
});
```

### cy.intercept

For mocking responses or verifying that the UI makes the right API calls:

```javascript
describe('Products page', () => {
  it('shows products when API returns data', () => {
    const products = productFactory.buildList(10);
    cy.intercept('GET', '/api/products', { body: products }).as('getProducts');
    cy.visit('/shop');
    cy.wait('@getProducts');
    cy.getByCy('product-card').should('have.length', 10);
  });

  it('shows error when API fails', () => {
    cy.intercept('GET', '/api/products', { statusCode: 500 }).as('getProducts');
    cy.visit('/shop');
    cy.wait('@getProducts');
    cy.getByCy('error-banner').should('contain', 'Unable to load');
  });

  it('shows empty state', () => {
    cy.intercept('GET', '/api/products', { body: [] }).as('getProducts');
    cy.visit('/shop');
    cy.wait('@getProducts');
    cy.getByCy('empty-state').should('be.visible');
  });

  it('verifies request payload', () => {
    cy.intercept('POST', '/api/orders').as('createOrder');
    cy.visit('/checkout');
    cy.getByCy('place-order').click();

    cy.wait('@createOrder').then((interception) => {
      expect(interception.request.body).to.have.property('items');
      expect(interception.response.statusCode).to.eq(201);
    });
  });
});
```

## 16. Authentication Patterns

### UI login only for the login test itself

```javascript
it('user can log in via UI', () => {
  cy.visit('/login');
  cy.getByCy('email').type(Cypress.env('TEST_EMAIL'));
  cy.getByCy('password').type(Cypress.env('TEST_PASSWORD'), { log: false });
  cy.getByCy('submit').click();
  cy.url().should('include', '/dashboard');
});
```

### cy.session for everything else

`cy.session` caches the auth state and reuses it. The first test in a run pays the cost of logging in everything after is fast. `cacheAcrossSpecs: true` is what makes this actually useful. Without it, you log in again at the start of every spec file.

```javascript
// cypress/support/commands.js
Cypress.Commands.add('login', (email, password) => {
  cy.session(
    [email, password],
    () => {
      cy.request('POST', `${Cypress.env('apiUrl')}/auth/login`, { email, password })
        .then(({ body }) => {
          window.localStorage.setItem('authToken', body.token);
        });
    },
    {
      validate() {
        cy.request({
          url: `${Cypress.env('apiUrl')}/auth/me`,
          headers: { Authorization: `Bearer ${window.localStorage.getItem('authToken')}` },
        }).its('status').should('eq', 200);
      },
      cacheAcrossSpecs: true,
    }
  );
});
```

```javascript
// In tests
beforeEach(() => {
  cy.login(Cypress.env('users').standard, Cypress.env('users').standardPassword);
  cy.visit('/dashboard');
});
```

## 17. Tagging Strategy

```bash
npm install --save-dev @cypress/grep
```

```javascript
// cypress/support/e2e.js
import '@cypress/grep';
require('@cypress/grep')();
```

The plugin also needs to be registered in `cypress.config.js` (already included in section 4).

Tagging tests:

```javascript
describe('Login', { tags: ['@regression'] }, () => {

  it('valid login redirects to dashboard', { tags: ['@smoke', '@critical'] }, () => {
    // ...
  });

  it('shows password reset link', { tags: ['@regression'] }, () => {
    // ...
  });

  it('handles SSO redirect', { tags: ['@regression', '@flaky'] }, () => {
    // ...
  });
});
```

Running by tag:

```bash
npx cypress run --env grepTags=@smoke
npx cypress run --env grepTags="@smoke @critical"
npx cypress run --env grepTags="@regression --@flaky"
```

The tag taxonomy I use:

| Tag | What it means | When it runs |
|-----|---------------|--------------|
| `@smoke` | Critical path, fast | Every PR, every deploy |
| `@regression` | Full functional coverage | Nightly, before releases |
| `@critical` | Revenue, security, compliance | Every PR and after deploy |
| `@flaky` | Known unstable, being investigated | Excluded from blocking pipelines |
| `@slow` | Over 30s | Separate pipeline |
| `@wip` | In progress, not ready | Excluded from CI |

Tag at both `describe` and `it` level. `describe` tags apply to every test inside use them for coverage classification. `it` tags add specifics like `@critical` or `@flaky`.

## 18. Mochawesome Reporting

Mochawesome generates HTML reports that stakeholders can actually read without needing to understand terminal output.

```bash
npm install --save-dev cypress-mochawesome-reporter
```

Configuration is already covered in section 4. Each run generates JSON per spec, and the `posttest` script merges them into a single HTML report at `cypress/reports/html/report.html`. Failure screenshots attach automatically.

For parallel runs where the auto-merge isn't enough:

```bash
npm install --save-dev mochawesome-merge mochawesome-report-generator
```

The `report:merge` and `report:generate` scripts handle this (already in the `package.json` setup from section 2).

Other options worth knowing: Allure (`@shelex/cypress-allure-plugin`) for trend graphs and richer analytics, JUnit XML (`mocha-junit-reporter`) if you're on Jenkins or Azure DevOps, and Cypress Cloud if you're using their parallelization anyway.

## 19. CI/CD Integration

```yaml
# .github/workflows/cypress.yml
name: Cypress Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  schedule:
    - cron: '0 2 * * *'

jobs:
  smoke:
    name: Smoke Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request' || github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      - name: Cypress smoke run
        uses: cypress-io/github-action@v6
        with:
          browser: chrome
          command: npm run cy:smoke
        env:
          CYPRESS_TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - name: Upload screenshots on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: cypress-screenshots
          path: cypress/screenshots
          retention-days: 7

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: mochawesome-report
          path: cypress/reports/html
          retention-days: 30

  regression:
    name: Regression Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run cy:regression
```

Run smoke on every PR, keep it under 5 minutes. Run regression nightly. Always upload screenshots and videos on failure, you'll need them. Cache `node_modules` or your pipeline times will be painful. Don't run the full regression on every commit; it'll slow feedback down to the point where people stop paying attention to it.

## 20. Parallelization

### With Cypress Cloud

```yaml
- run: npm run cy:run -- --record --parallel --key $CYPRESS_RECORD_KEY
```

Easiest option if you're already paying for it.

### Without Cypress Cloud

```bash
npm install --save-dev cypress-split
```

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx cypress run --env split=true,splitIndex=${{ matrix.shard }},splitTotal=4
```

When to bother: if the suite is taking more than 10 minutes serially, parallelization is worth it. Under 5 minutes, don't add the complexity yet.

## 21. Debugging

### In the test runner

The time-travel debugger is genuinely useful. Hover over any command in the command log to see the DOM state at that exact step. The Selector Playground helps when you're not sure what `data-cy` to use. The network panel shows all intercepted requests.

Useful commands:

```javascript
cy.debug();                       // pauses with devtools open
cy.pause();                       // pauses interactively
cy.log('checkpoint');
cy.task('log', 'shows in terminal');

cy.getByCy('thing').then(($el) => {
  console.log($el.css('display'));
  debugger;
});
```

### Diagnosing flakiness

When a test fails sometimes but not always:

```
Same machine, same browser?
  Yes → likely a race condition or timing issue
    → is there a missing assertion before the action?
    → is there a hardcoded wait that sometimes isn't long enough?
    → is there a CSS animation that causes the element to be briefly non-interactive?
  No → environmental difference
    → check baseUrl, env vars, viewport size

Fails in CI but not locally?
  → CI machines are slower; consider bumping timeouts specifically for CI
  → CI is usually Linux; you're probably on Mac (font rendering, locale differences)
  → CI runs headless; reproduce locally with --headed
  → CI might not have the test data; make sure you're seeding it explicitly
```

### When cy.wait is acceptable

```javascript
// Almost always wrong
cy.wait(3000);

// Right wait for an intercepted request
cy.intercept('GET', '/api/data').as('getData');
cy.visit('/page');
cy.wait('@getData');

// Righ wait for an element state change
cy.getByCy('loading').should('not.exist');
cy.getByCy('content').should('be.visible');

// Acceptable animation with no observable end state
cy.getByCy('animated').click();
cy.wait(300);   // matches a 250ms CSS transition
```

## 22. Best Practices

These aren't rules I follow because someone told me to. They're things I've broken and had to fix.

**Test isolation.** Every test must pass alone with `it.only`. Every test must pass with the full suite. No test should depend on another running first. When tests have hidden order dependencies, they're fine on Monday and broken on Wednesday because someone added a new test above.

**State management.** Use `cy.session` for auth and never route every test through the login UI. Seed test data via API or task, not via UI clicks. If your test creates data, clean it up afterwards.

**Locators.** Always prefer `data-cy`. Never use `nth-child`, deeply nested CSS, or XPath unless you genuinely have no better option and if you don't, leave a comment explaining why.

**Assertions.** Every test has at least one `should`. A test without assertions isn't testing anything; it's just exercising the UI. Assert on user-visible behavior, not implementation details.

**Waits.** Never `cy.wait(arbitrary_ms)`. Wait on intercepts or assertions. Don't raise `defaultCommandTimeout` above 10s to fix a failing test find out why the test is slow.

**Naming.** A test name should describe behavior clearly enough that someone who didn't write it knows what failed:

```javascript
// Not useful
it('test login');

// Useful
it('redirects to dashboard after valid login');
it('shows lockout error after 5 failed attempts');
```

**Sensitive data.** Passwords and tokens in environment variables only. `{ log: false }` on any `cy.type` call for credentials. Never commit `.env`.

## 23. Code Review Checklist

Use this before approving a PR with test changes:

- Test name describes behavior clearly
- Test passes in isolation (`it.only`)
- Locators use `data-cy` (or exception is documented)
- No hardcoded `cy.wait(ms)` without justification
- Assertions are meaningful and user-facing
- No `{ force: true }` without a clear reason
- Repeated logic is extracted to custom commands (3+ uses)
- Test data comes from fixtures/factories, not inline strings
- Tags are correct (`@smoke`, `@regression`, etc.)
- No new `@flaky` tag without a linked ticket
- No credentials or secrets are committed
- Negative path checked: does the test fail when the feature breaks?

## 24. Maintenance

A test suite isn't something you write and leave. I budget around 25% of testing time for maintenance. It sounds like a lot until you've been on a team that doesn't and watched the suite slowly stop meaning anything.

Weekly checklist:

- Review CI failures from the week
- Fix flakes or move unstable tests to `@flaky`
- Update Page Objects affected by UI changes
- Remove tests for deleted features

Monthly checklist:

- Review all `@flaky` tests (fix or remove)
- Audit slow tests and optimize critical ones
- Run `npm outdated` and plan dependency updates
- Refactor repeated patterns into commands/helpers

Quarterly checklist:

- Revisit tag taxonomy and CI usage
- Update factories to reflect current domain fields
- Remove unused fixtures, helpers, and dead specs
- Re-baseline timing targets for smoke/regression

Metrics I track:

| Metric | Target |
|--------|--------|
| Smoke suite duration | Under 5 min |
| Full regression duration | Under 30 min |
| Flake rate | Under 1% |
| Pass rate over 30 days | Over 95% |

## 25. Useful Plugins

| Plugin | What it does |
|--------|-------------|
| `cypress-mochawesome-reporter` | HTML reports with screenshots |
| `@cypress/grep` | Tag-based test filtering |
| `@faker-js/faker` | Dynamic test data |
| `@testing-library/cypress` | Accessible queries |
| `cypress-axe` | Accessibility violation detection |
| `cypress-real-events` | Native browser events (drag, hover) |
| `cypress-iframe` | iframe-friendly commands |
| `cypress-image-snapshot` | Visual regression |
| `@cypress/code-coverage` | Code coverage alongside E2E |
| `cypress-split` | Free parallelization without Cypress Cloud |

## 26. Closing Thoughts

A few things I'd want a teammate to know before diving in.

Tests are communication. A failing test should tell anyone dev, PM, support engineer exactly what behavior broke and why it matters. If the test name is "test login 2," that's already a failure.

Speed matters more than you'd expect. A 45-minute suite nobody runs is worse than no suite. Keep smoke fast, keep regression reasonable. If the suite is slow, people stop running it locally and stop trusting it in CI.

Trust is the thing that's hard to rebuild. One reliably flaky test in CI starts a culture of "just rerun it." That culture spreads. Two weeks in and developers have stopped reading test failures altogether. Fixing a flaky test is never just about one test.

The framework is a product your teammates use. Treat the README, custom commands, and setup docs as you would any tool you'd want to actually use. If it's annoying to work with, people will route around it.

Automation handles the repetitive 80%. The other 20% the weird flows, the accessibility gaps, the edge cases that live in user behavior that's the part you can't automate your way out of. The point of a good Cypress suite is to free up time and attention for the testing that actually needs a human.
