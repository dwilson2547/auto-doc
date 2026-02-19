# Gherkin: A Guide to Behavior-Driven Development Syntax

Gherkin is a plain-text language used to describe software behavior in a structured, human-readable format. It serves as the backbone of **Behavior-Driven Development (BDD)**, bridging communication between developers, testers, and non-technical stakeholders. Gherkin files (`.feature` files) are used by frameworks such as Cucumber, Behave, SpecFlow, and Behat.

## Table of Contents

- [What is Gherkin?](#what-is-gherkin)
- [Core Syntax](#core-syntax)
  - [Feature](#feature)
  - [Scenario](#scenario)
  - [Steps: Given, When, Then, And, But](#steps-given-when-then-and-but)
  - [Background](#background)
  - [Scenario Outline and Examples](#scenario-outline-and-examples)
  - [Tags](#tags)
  - [Doc Strings](#doc-strings)
  - [Data Tables](#data-tables)
  - [Comments](#comments)
- [Syntax Examples](#syntax-examples)
  - [Basic Login Feature](#basic-login-feature)
  - [Shopping Cart Feature](#shopping-cart-feature)
  - [API Feature with Data Tables](#api-feature-with-data-tables)
  - [Scenario Outline Example](#scenario-outline-example)
- [Use Cases](#use-cases)
  - [Web Application Testing](#web-application-testing)
  - [API Testing](#api-testing)
  - [Business Rule Validation](#business-rule-validation)
  - [Acceptance Testing](#acceptance-testing)
  - [Regression Testing](#regression-testing)
- [Best Practices](#best-practices)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)
- [Gherkin with Popular Frameworks](#gherkin-with-popular-frameworks)
- [Additional Resources](#additional-resources)

---

## What is Gherkin?

Gherkin uses a set of special **keywords** to give structure and meaning to executable specifications. Its design goals are:

- **Human-readable**: Non-developers can read and write scenarios
- **Executable**: Scenarios are directly tied to automated test steps
- **Living documentation**: Feature files serve as always-up-to-date documentation

Gherkin is whitespace-sensitive and uses indentation to define the structure of a feature file.

---

## Core Syntax

### Feature

The `Feature` keyword opens a feature file and provides a high-level description of the functionality being tested. Everything under the `Feature` line is part of that feature.

```gherkin
Feature: User Authentication
  As a registered user
  I want to log in to my account
  So that I can access my personal dashboard
```

The optional narrative lines after `Feature:` (the "As a / I want / So that" format) follow the **User Story** pattern and help communicate intent.

---

### Scenario

A `Scenario` describes a single concrete example of the feature's behavior. Each scenario is independent and should be able to run in isolation.

```gherkin
Scenario: Successful login with valid credentials
  Given I am on the login page
  When I enter valid credentials
  Then I should be redirected to my dashboard
```

---

### Steps: Given, When, Then, And, But

Steps are the building blocks of scenarios. Each step begins with one of five keywords:

| Keyword | Purpose |
|---------|---------|
| `Given` | Sets up the initial context or preconditions |
| `When`  | Describes the action or event that triggers behavior |
| `Then`  | Describes the expected outcome or result |
| `And`   | Continues a `Given`, `When`, or `Then` with another step of the same type |
| `But`   | Similar to `And`, typically used for negative assertions |

```gherkin
Scenario: User adds item to cart
  Given I am logged in as a customer
  And I am viewing the product catalog
  When I click "Add to Cart" on "Wireless Headphones"
  And I proceed to checkout
  Then my cart should contain 1 item
  But my wishlist should remain empty
```

> **Tip:** `And` and `But` take on the meaning of the preceding keyword (`Given`, `When`, or `Then`). They exist purely for readability.

---

### Background

`Background` allows you to define steps that are common to **all** scenarios in a feature file. It runs before every scenario, eliminating repetition.

```gherkin
Feature: Account Management

  Background:
    Given I am logged in as an admin user
    And the database is populated with test data

  Scenario: View all users
    When I navigate to the Users section
    Then I should see a list of all registered users

  Scenario: Delete a user
    When I delete the user "john.doe@example.com"
    Then the user should no longer appear in the list
```

---

### Scenario Outline and Examples

`Scenario Outline` (also called `Scenario Template` in some frameworks) lets you run the same scenario with multiple sets of data using an `Examples` table. Placeholders in the scenario are wrapped in angle brackets `< >`.

```gherkin
Scenario Outline: Login with different user roles
  Given I am on the login page
  When I log in as "<role>" with password "<password>"
  Then I should see the "<dashboard>" dashboard

  Examples:
    | role    | password    | dashboard |
    | admin   | admin123    | Admin     |
    | manager | manager456  | Manager   |
    | viewer  | viewer789   | Read-Only |
```

Each row in `Examples` produces a separate test execution.

---

### Tags

Tags are annotations prefixed with `@` placed above a `Feature`, `Scenario`, or `Scenario Outline`. They are used for:
- Filtering which tests to run
- Categorizing tests (e.g., smoke, regression, wip)
- Controlling hooks (e.g., run setup only for `@database` tests)

```gherkin
@smoke @authentication
Feature: User Login

  @happy-path
  Scenario: Successful login
    Given I am on the login page
    When I enter valid credentials
    Then I am logged in

  @negative @wip
  Scenario: Login with invalid password
    Given I am on the login page
    When I enter an incorrect password
    Then I see an error message
```

Run only tagged scenarios (example with Cucumber CLI):
```bash
cucumber --tags @smoke
cucumber --tags "@smoke and not @wip"
```

---

### Doc Strings

Doc Strings allow you to pass a larger block of text to a step, useful for multi-line content like JSON payloads, XML, or email bodies. They are delimited by triple double-quotes `"""`.

```gherkin
Scenario: Submit a JSON API request
  Given the request body is:
    """
    {
      "username": "alice",
      "email": "alice@example.com",
      "role": "editor"
    }
    """
  When I POST to "/api/users"
  Then the response status should be 201
```

---

### Data Tables

Data Tables allow you to pass structured tabular data directly to a step. Unlike `Examples`, they are used within a single step rather than to multiply scenarios.

```gherkin
Scenario: Create multiple products in bulk
  Given the following products exist in the catalog:
    | name                | category    | price  | in_stock |
    | Wireless Mouse      | Electronics | 29.99  | true     |
    | Mechanical Keyboard | Electronics | 89.99  | true     |
    | USB-C Hub           | Electronics | 49.99  | false    |
  When I filter by "in_stock" equals "true"
  Then I should see 2 products in the results
```

---

### Comments

Lines starting with `#` are comments and are ignored by the parser.

```gherkin
# This scenario validates the password reset flow
Scenario: Request a password reset
  Given I am on the login page
  # Enter an email that is registered in the system
  When I click "Forgot Password" and enter "user@example.com"
  Then I should receive a password reset email
```

---

## Syntax Examples

### Basic Login Feature

```gherkin
Feature: User Authentication
  As a registered user
  I want to securely log in and out of the application
  So that my account remains protected

  Background:
    Given the application is running
    And the database contains user "jane.doe@example.com" with password "SecurePass1!"

  @smoke
  Scenario: Successful login with valid credentials
    Given I am on the login page
    When I enter email "jane.doe@example.com" and password "SecurePass1!"
    And I click the "Login" button
    Then I should be redirected to "/dashboard"
    And I should see the welcome message "Welcome back, Jane!"

  @negative
  Scenario: Login fails with incorrect password
    Given I am on the login page
    When I enter email "jane.doe@example.com" and password "WrongPassword"
    And I click the "Login" button
    Then I should see the error "Invalid email or password"
    And I should remain on the login page

  Scenario: User logs out successfully
    Given I am logged in as "jane.doe@example.com"
    When I click the "Logout" button
    Then I should be redirected to the login page
    And my session should be invalidated
```

---

### Shopping Cart Feature

```gherkin
Feature: Shopping Cart Management
  As an online shopper
  I want to manage items in my cart
  So that I can review and adjust my order before purchasing

  Background:
    Given I am logged in as a customer
    And the store has the following items available:
      | item                | price  | stock |
      | Running Shoes       | 120.00 | 10    |
      | Sports Water Bottle | 25.00  | 50    |
      | Gym Bag             | 75.00  | 5     |

  Scenario: Add a single item to the cart
    When I add "Running Shoes" to my cart
    Then my cart should contain 1 item
    And the cart total should be "$120.00"

  Scenario: Add multiple items to the cart
    When I add "Running Shoes" to my cart
    And I add "Sports Water Bottle" to my cart
    Then my cart should contain 2 items
    And the cart total should be "$145.00"

  Scenario: Remove an item from the cart
    Given my cart contains "Running Shoes" and "Gym Bag"
    When I remove "Gym Bag" from my cart
    Then my cart should contain 1 item
    And the cart total should be "$120.00"

  Scenario: Cart is empty at checkout attempt
    Given my cart is empty
    When I navigate to the checkout page
    Then I should see the message "Your cart is empty"
    And the "Place Order" button should be disabled
```

---

### API Feature with Data Tables

```gherkin
Feature: User Management API
  As a system administrator
  I want to manage users via the REST API
  So that I can automate user provisioning

  @api @smoke
  Scenario: Create a new user via API
    Given the API is available at "https://api.example.com"
    And I have a valid authentication token
    When I send a POST request to "/users" with the body:
      """
      {
        "firstName": "Alice",
        "lastName": "Smith",
        "email": "alice.smith@example.com",
        "role": "viewer"
      }
      """
    Then the response status code should be 201
    And the response body should contain "userId"
    And the response body should include:
      | field     | value                      |
      | firstName | Alice                      |
      | lastName  | Smith                      |
      | email     | alice.smith@example.com    |
      | role      | viewer                     |

  @api
  Scenario: Retrieve a list of users with pagination
    Given 25 users exist in the system
    When I send a GET request to "/users?page=1&limit=10"
    Then the response status code should be 200
    And the response should contain 10 users
    And the response should include pagination metadata with "totalCount" of 25

  @api @negative
  Scenario: Creating a user with a duplicate email returns an error
    Given a user with email "existing@example.com" already exists
    When I send a POST request to "/users" with email "existing@example.com"
    Then the response status code should be 409
    And the error message should be "A user with this email already exists"
```

---

### Scenario Outline Example

```gherkin
Feature: Password Validation
  As the security team
  I want to enforce strong password rules
  So that user accounts are protected

  Scenario Outline: Validate password strength
    Given I am on the registration page
    When I enter the password "<password>"
    Then I should see the validation message "<message>"

    Examples: Weak passwords
      | password | message                                      |
      | 123      | Password must be at least 8 characters long  |
      | password | Password must contain at least one number    |
      | pass1    | Password must be at least 8 characters long  |

    Examples: Strong passwords
      | password       | message         |
      | Secure1!       | Password is strong |
      | MyP@ssw0rd     | Password is strong |
      | C0mplex#Pass!  | Password is strong |
```

---

## Use Cases

### Web Application Testing

Gherkin excels at describing end-to-end user flows in web applications. Teams use it to validate:
- User registration and authentication
- Navigation and page rendering
- Form submissions and validations
- Role-based access control
- Checkout and payment flows

**Example**: An e-commerce team writes Gherkin features for every user-facing workflow. QA engineers, product owners, and developers review the same `.feature` files, ensuring a shared understanding before any code is written.

---

### API Testing

Gherkin is well-suited for documenting and testing RESTful or GraphQL APIs:
- Verifying HTTP status codes and response bodies
- Testing authentication and authorization headers
- Validating request payloads and schema conformance
- Testing error handling and edge cases

**Example**: A backend team uses Gherkin with RestAssured (Java) or `requests` (Python/Behave) to run automated API contract tests on every pull request.

---

### Business Rule Validation

Complex business logic — such as pricing rules, discount calculations, or eligibility checks — can be expressed clearly in Gherkin, making it easy for business analysts to verify the rules are correct.

```gherkin
Feature: Discount Eligibility
  Scenario Outline: Apply discounts based on membership tier
    Given a customer has a "<tier>" membership
    When they purchase items totaling "$<amount>"
    Then their discount should be "<discount>%"

    Examples:
      | tier     | amount | discount |
      | Bronze   | 100    | 5        |
      | Silver   | 100    | 10       |
      | Gold     | 100    | 15       |
      | Platinum | 100    | 20       |
```

---

### Acceptance Testing

Gherkin scenarios written before development serve as **acceptance criteria**. A feature is considered "done" only when all associated scenarios pass. This is the foundation of the BDD workflow:

1. **Discuss**: Stakeholders, developers, and testers collaborate to write scenarios
2. **Develop**: Developers write code to make scenarios pass
3. **Deliver**: Passing scenarios confirm the feature meets acceptance criteria

---

### Regression Testing

Gherkin feature files accumulate over time, forming a comprehensive regression suite. Tags make it easy to run subsets:

```bash
# Run only smoke tests before a production deploy
cucumber --tags @smoke

# Run full regression suite nightly
cucumber --tags @regression

# Skip tests still in progress
cucumber --tags "not @wip"
```

---

## Best Practices

1. **Write scenarios from the user's perspective** — focus on *what* the system does, not *how* it does it.

2. **Keep scenarios short and focused** — each scenario should test one specific behavior. Aim for 3–7 steps per scenario.

3. **Use declarative language** — prefer `When I log in as an admin` over `When I enter "admin" in the username field and "password123" in the password field and click the Login button`.

4. **Avoid implementation details** — steps should describe observable behavior, not UI elements or code internals.

5. **Use `Background` wisely** — only include steps in `Background` if they truly apply to *every* scenario in the file.

6. **Keep `Examples` tables readable** — use meaningful column names and realistic data values.

7. **Tag strategically** — use a consistent tagging convention across your team (e.g., `@smoke`, `@regression`, `@wip`, `@negative`).

8. **Treat feature files as documentation** — keep them updated when requirements change; outdated scenarios are worse than no scenarios.

9. **One feature per file** — keep each `.feature` file focused on a single feature or user story.

10. **Collaborate on scenarios** — write Gherkin in a **Three Amigos** session with a developer, tester, and business representative.

---

## Common Mistakes to Avoid

| Mistake | Problem | Better Approach |
|---------|---------|-----------------|
| Mixing UI details in steps | Brittle tests that break on UI changes | Use abstract, behavior-focused language |
| Very long scenarios | Hard to read and maintain | Break into smaller, focused scenarios |
| Sharing state between scenarios | Tests become order-dependent | Use `Background` or hooks; keep scenarios independent |
| Testing multiple behaviors in one scenario | Unclear failure point | One scenario = one behavior |
| Using `And` as the first step | Confusing without context | Always start with `Given`, `When`, or `Then` |
| Vague step descriptions | Ambiguous test intent | Use specific, unambiguous language |

---

## Gherkin with Popular Frameworks

| Framework | Language | Description |
|-----------|----------|-------------|
| [Cucumber](https://cucumber.io/) | Java, JavaScript, Ruby, and more | The most widely used BDD framework |
| [Behave](https://behave.readthedocs.io/) | Python | Popular BDD framework for Python projects |
| [SpecFlow](https://specflow.org/) | C# / .NET | BDD for the .NET ecosystem |
| [Behat](https://behat.org/) | PHP | BDD framework for PHP |
| [Godog](https://github.com/cucumber/godog) | Go | Cucumber-style BDD for Go |
| [pytest-bdd](https://pytest-bdd.readthedocs.io/) | Python | Gherkin integration for pytest |

All of these frameworks parse `.feature` files written in Gherkin and map each step to a corresponding step definition written in the host language.

**Example step definition (Python / Behave):**

```python
from behave import given, when, then

@given('I am on the login page')
def step_navigate_to_login(context):
    context.browser.visit('/login')

@when('I enter email "{email}" and password "{password}"')
def step_enter_credentials(context, email, password):
    context.browser.fill('email', email)
    context.browser.fill('password', password)

@then('I should be redirected to "{path}"')
def step_verify_redirect(context, path):
    assert context.browser.url.endswith(path)
```

---

## Additional Resources

- [Official Gherkin Reference](https://cucumber.io/docs/gherkin/reference/)
- [Cucumber Documentation](https://cucumber.io/docs/)
- [Behave Documentation](https://behave.readthedocs.io/)
- [SpecFlow Documentation](https://docs.specflow.org/)
- [BDD in Action (Book)](https://www.manning.com/books/bdd-in-action-second-edition)
- [The Three Amigos — Agile Alliance](https://www.agilealliance.org/glossary/three-amigos/)
- [Writing Good Gherkin — Automation Panda](https://automationpanda.com/2017/01/30/bdd-101-writing-good-gherkin/)
