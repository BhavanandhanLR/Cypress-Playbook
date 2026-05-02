# Cypress Cheatsheet (The One I Actually Use)

I keep this open while writing tests. This is not every Cypress command on earth. It is the stuff I reach for daily.

## 1) Locators I Trust

If you can add attributes in the app, use data attributes. They survive UI refactors better than class selectors.

```javascript
cy.get('[data-cy="submit-btn"]')
cy.get('[data-testid="submit-btn"]')
cy.get('[data-test="submit-btn"]')

// with custom command
cy.getByCy('submit-btn')
```

Good fallback options:

```javascript
cy.get('[name="email"]')
cy.get('[type="submit"]')
cy.get('[aria-label="Close"]')
cy.contains('button', 'Save')
```

Try to avoid brittle selectors like deep nesting:

```javascript
// fragile
cy.get('div > div > div > span:nth-child(3)')
```

Scoped selection is cleaner when the page is busy:

```javascript
cy.get('form').within(() => {
  cy.get('input[name="email"]').type('user@test.com')
  cy.contains('button', 'Sign in').click()
})
```

Useful traversal when needed:

```javascript
cy.get('input').closest('.field')
cy.get('input').parent()
cy.get('li').eq(2)
cy.get('li').first()
cy.get('li').last()
```

## 2) Actions I Use Most

### Navigation

```javascript
cy.visit('/login')
cy.go('back')
cy.reload()
```

### Typing

```javascript
cy.get('input[name="email"]').type('user@test.com')
cy.get('input[name="password"]').type('secret', { log: false })
cy.get('input[name="email"]').clear().type('new@test.com')
cy.get('input[name="search"]').type('laptop{enter}')
```

### Clicking

```javascript
cy.contains('button', 'Save').click()
cy.get('[data-cy="submit"]').click()
cy.get('[data-cy="submit"]').click({ force: true }) // only if you know why
cy.get('[data-cy="menu-item"]').rightclick()
```

### Checkbox / Radio / Select

```javascript
cy.get('[type="checkbox"]').check()
cy.get('[type="checkbox"]').uncheck()
cy.get('[type="radio"][value="yes"]').check()
cy.get('select[name="country"]').select('Ireland')
```

### File upload

```javascript
cy.get('input[type="file"]').selectFile('cypress/fixtures/photo.png')
```

### Scroll / Focus

```javascript
cy.get('footer').scrollIntoView()
cy.get('input[name="email"]').focus()
cy.focused().should('have.attr', 'name', 'email')
```

## 3) Assertions That Keep Tests Stable

I prefer clear assertions over fancy chains.

```javascript
cy.get('[data-cy="toast"]').should('be.visible')
cy.get('[data-cy="error"]').should('not.exist')
cy.get('[data-cy="title"]').should('have.text', 'Dashboard')
cy.get('[data-cy="title"]').should('contain.text', 'Dash')
cy.get('input[name="email"]').should('have.value', 'user@test.com')
cy.get('button[type="submit"]').should('be.disabled')
cy.url().should('include', '/dashboard')
cy.title().should('include', 'Dashboard')
```

Count checks:

```javascript
cy.get('[data-cy="product-card"]').should('have.length', 6)
```

Chained style when it helps readability:

```javascript
cy.get('[data-cy="badge"]')
  .should('be.visible')
  .and('have.text', '3')
```

## 4) Waiting (Without Guessing)

Do not sleep unless there is absolutely no observable state to wait for.

```javascript
// avoid when possible
cy.wait(3000)

// preferred
cy.intercept('GET', '/api/data').as('getData')
cy.visit('/page')
cy.wait('@getData').its('response.statusCode').should('eq', 200)

// also good
cy.get('[data-cy="loader"]').should('not.exist')
cy.get('[data-cy="content"]').should('be.visible')
```

## 5) API and Network

### Direct API calls

```javascript
cy.request('GET', '/api/users')
cy.request('POST', '/api/users', { name: 'Alice' })
```

With explicit request object:

```javascript
cy.request({
  method: 'POST',
  url: '/api/login',
  body: { email: 'a@b.com', password: 'pass' },
  failOnStatusCode: false
}).then((response) => {
  expect(response.status).to.eq(200)
  expect(response.body).to.have.property('token')
})
```

### Intercept patterns

```javascript
// mock
cy.intercept('GET', '/api/users', { fixture: 'users.json' }).as('getUsers')

// spy
cy.intercept('POST', '/api/orders').as('createOrder')

cy.wait('@createOrder').then((interception) => {
  expect(interception.request.body).to.have.property('items')
})
```

## 6) Fixtures, Hooks, and Aliases

### Fixtures

```javascript
cy.fixture('users').then((users) => {
  cy.get('input[name="email"]').type(users.standard.email)
})
```

### Hooks

```javascript
describe('Auth', () => {
  beforeEach(() => {
    cy.visit('/login')
  })

  it('logs in', () => {
    // test
  })
})
```

### Aliases

```javascript
cy.get('input[name="email"]').as('email')
cy.get('@email').type('user@test.com')

cy.intercept('GET', '/api/profile').as('getProfile')
cy.wait('@getProfile')
```

## 7) Custom Commands I Reuse

In cypress/support/commands.js:

```javascript
Cypress.Commands.add('getByCy', (selector) => {
  return cy.get(`[data-cy="${selector}"]`)
})

Cypress.Commands.add('loginByApi', (email, password) => {
  cy.request('POST', '/api/login', { email, password }).then(({ body }) => {
    window.localStorage.setItem('token', body.token)
  })
})
```

Usage:

```javascript
cy.loginByApi('user@test.com', 'password123')
cy.visit('/dashboard')
```

## 8) Session Caching

This is one of the best speed wins.

```javascript
cy.session(
  [email, password],
  () => {
    cy.request('POST', '/api/login', { email, password }).then(({ body }) => {
      window.localStorage.setItem('token', body.token)
    })
  },
  {
    validate() {
      cy.request('/api/me').its('status').should('eq', 200)
    },
    cacheAcrossSpecs: true
  }
)
```

## 9) Debugging Quickly

```javascript
cy.pause()
cy.debug()
cy.log('checkpoint')
cy.screenshot('after-submit')

cy.get('[data-cy="price"]').then(($el) => {
  console.log($el.text())
  debugger
})
```

## 10) Run Commands

```bash
npx cypress open
npx cypress run
npx cypress run --headed
npx cypress run --browser chrome
npx cypress run --spec "cypress/e2e/auth/login.cy.js"
npx cypress run --env grepTags=@smoke
npx cypress run --config baseUrl=https://staging.example.com
```

## 11) Copy-Paste Patterns

### Login then test page

```javascript
beforeEach(() => {
  cy.loginByApi('user@test.com', 'password123')
  cy.visit('/dashboard')
})

it('shows user name', () => {
  cy.get('[data-cy="user-name"]').should('contain', 'Alice')
})
```

### Form submit flow

```javascript
cy.get('[data-cy="email"]').type('user@test.com')
cy.get('[data-cy="password"]').type('pass123', { log: false })
cy.get('[data-cy="terms"]').check()
cy.get('[data-cy="country"]').select('Ireland')
cy.get('[data-cy="submit"]').click()
cy.url().should('include', '/welcome')
```

### Optional popup handling

```javascript
cy.get('body').then(($body) => {
  if ($body.find('[data-cy="cookie-banner"]').length) {
    cy.get('[data-cy="accept-cookies"]').click()
  }
})
```

## 12) Quick Reminders

- Prefer data-cy selectors over classes.
- Avoid force click unless you can explain why.
- Avoid fixed waits when you can wait on state.
- Keep tests independent. They should pass alone.
- A short, clear assertion beats a clever one.

This cheatsheet is intentionally opinionated. Update it when your team patterns change.
