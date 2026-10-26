# Architecture — UniversalTestFramework

-----

## Overview

UniversalTestFramework is a multi-module Maven project designed around separation of concerns. The core framework is completely decoupled from any specific application under test, allowing new test projects to be onboarded by adding a module and consuming the core as a dependency.

-----

## Module Dependency Graph

```
pom.xml (parent)
├── core-framework
│   └── (no internal dependencies)
├── shared-domain-objects
│   └── depends on: core-framework
├── shared-resources
│   └── (configuration and data files only)
└── test-projects/
    └── parabank-tests
        └── depends on: core-framework, shared-domain-objects, shared-resources
```

-----

## Module Responsibilities

### core-framework

The reusable heart of the framework. Contains no application-specific code.

|Package        |Responsibility                                           |
|---------------|---------------------------------------------------------|
|`base`         |BaseTest, WebTestBase, ApiTestBase — lifecycle management|
|`config`       |ConfigManager — 4-layer property resolution              |
|`drivers`      |DriverManager — thread-safe WebDriver management         |
|`listeners`    |RetryAnalyzer, RetryTransformer, TestListener            |
|`utilities`    |WaitUtils, ScreenshotUtils, DataGenerator, JSUtils, etc. |
|`reporting`    |ExtentManager, ReportManager, AllureHelper               |
|`api`          |BaseAPIClient, RestClient, ResponseValidator             |
|`exceptions`   |FrameworkException, ElementNotFoundException             |
|`dataproviders`|CsvDataProvider, JsonDataProvider                        |

### shared-domain-objects

Shared models and utilities that can be reused across multiple test projects.

|Package     |Responsibility                                 |
|------------|-----------------------------------------------|
|`models`    |User, Account, ApiResponse, banking models     |
|`pages`     |CommonBasePage, CommonLoginPage                |
|`components`|HeaderComponent, ModalComponent                |
|`constants` |ErrorConstants, MessageConstants, TimeConstants|
|`utils`     |DateUtils, StringUtils, ValidationUtils        |

### shared-resources

Static resources shared across all test projects.

|Directory      |Contents                              |
|---------------|--------------------------------------|
|`data/common`  |Shared CSV data files                 |
|`reporting`    |Allure and ExtentReports configuration|
|`templates`    |Email and report templates            |
|`testng-backup`|TestNG suite XML backups              |

### parabank-tests

The primary test project implementing the framework against the ParaBank banking application.

|Package        |Responsibility                           |
|---------------|-----------------------------------------|
|`pages`        |Page Object Model for all ParaBank pages |
|`components`   |LoginForm, AccountServicesPanel          |
|`api`          |REST clients for ParaBank API endpoints  |
|`models`       |BillPayee, RegistrationUser, Transaction |
|`actions`      |Action layer for hybrid test patterns    |
|`dto`          |Request/response objects, runtime DTOs   |
|`managers`     |PageObjectManager, SessionManager        |
|`utils`        |ParabankConfigLoader, ServerHealthChecker|
|`tests`        |Test classes (UI, API, Hybrid, E2E)      |
|`dataproviders`|BillPayeeDataProvider, LoginDataProvider |

-----

## Configuration Architecture

ConfigManager resolves properties in the following priority order:

```
1. System Properties     (-Dproperty=value via Maven/CLI)
2. local.properties      (src/test/resources/local.properties — gitignored)
3. Environment config    (config/parabank-{env}.properties)
4. Framework defaults    (framework-defaults.properties)
```

This allows:

- Local developer overrides via `local.properties`
- CI/CD environment selection via `-Denv=prod`
- Sensible defaults without any configuration

-----

## Base Class Hierarchy

```
BaseTest
└── WebTestBase
    └── ParaBankUIBaseTest
        └── (all UI test classes)

BaseTest
└── ApiTestBase
    └── ParaBankAPIBaseTest
        └── (all API test classes)
```

### BaseTest responsibilities

- WebDriver initialisation and teardown
- Configuration loading
- Screenshot on failure
- TestNG lifecycle hooks (@BeforeSuite, @BeforeMethod, @AfterMethod)

### WebTestBase responsibilities

- Browser window management
- Timeout configuration
- Navigation utilities
- Cookie and storage management

### ParaBankUIBaseTest responsibilities

- ParaBank-specific setup (login, page objects)
- Environment URL resolution
- Logout cleanup after each test

-----

## RetryAnalyzer Architecture

The RetryAnalyzer is registered via Java ServiceLoader, making it automatically applied to all tests without requiring suite XML or `@Listeners` annotations.

```
META-INF/services/org.testng.ITestNGListener
    → RetryTransformer (IAnnotationTransformer)
        → injects RetryAnalyzer into every @Test annotation
            → RetryAnalyzer.retry() checks config and retries on failure
```

Key design decisions:

- **ServiceLoader** instead of suite XML — avoids conflicts with group-based test selection strategy
- **DisabledRetryAnalyzer detection** — TestNG 7.x never returns null from `getRetryAnalyzerClass()`, returns `DisabledRetryAnalyzer` instead; the transformer checks for this class by name
- **Config-driven** — retry count and enabled flag controlled via `local.properties`

-----

## Test Execution Strategy

Tests are selected using TestNG groups, not suite XML files. This is intentional:

```bash
# Run by group via Maven profile
mvn test -pl test-projects/parabank-tests -P smoke
mvn test -pl test-projects/parabank-tests -P regression
mvn test -pl test-projects/parabank-tests -P api
```

Groups used:

|Group       |Description                   |
|------------|------------------------------|
|`smoke`     |Core happy-path scenarios     |
|`sanity`    |Extended happy-path validation|
|`regression`|Full regression suite         |
|`api`       |REST API tests only           |
|`hybrid`    |API setup + UI verification   |
|`e2e`       |Full end-to-end user journeys |

-----

## Hybrid Test Pattern

The hybrid pattern is the primary differentiator of this framework. Rather than setting up test state through the UI (slow, fragile), hybrid tests use the API for setup and the UI only for verification.

```
Hybrid Test Flow:
1. Login via API        → obtain JSESSIONID
2. Create account via API → obtain account number
3. Verify account in UI → assert account visible on dashboard
```

This approach is faster, more reliable, and demonstrates real-world test engineering thinking.

-----

## CI/CD Pipeline

```
push to main
    └── Build (mvn install -DskipTests)
        ├── API Tests (no browser)
        ├── UI Smoke Tests (headless Chrome)
        └── Hybrid Tests (headless Chrome)
```

All jobs upload surefire reports as artifacts for post-run analysis.

-----