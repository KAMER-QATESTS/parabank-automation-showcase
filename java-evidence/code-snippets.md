# Java Automation — Code Snippets

Full source code is in the private repository.
These snippets demonstrate key architectural decisions in the framework.

---

## 1. Thread-Safe WebDriver with ThreadLocal

Each parallel test thread gets its own isolated browser instance.
Calling `remove()` after `quit()` prevents memory leaks in thread pools.

```java
public class DriverManager {

    private static final ThreadLocal<WebDriver> driver = new ThreadLocal<>();

    public static WebDriver getDriver() {
        return driver.get();
    }

    public static void initDriver(String browser, boolean headless) {
        WebDriver webDriver = DriverFactory.createDriver(browser, headless);
        driver.set(ThreadGuard.protect(webDriver));
    }

    public static void quitDriver() {
        if (driver.get() != null) {
            driver.get().quit();
            driver.remove(); // Prevents memory leak in thread pools
        }
    }
}
```

## 2. RetryAnalyzer — Environment Exceptions Only
Retries only on infrastructure failures (timeout, stale element).
Never retries AssertionError — a genuine defect should not be masked
or, in a financial application, executed multiple times.

```java
@Override
public boolean retry(ITestResult result) {
    if (!retryEnabled) return false;

    Throwable cause = result.getThrowable();
    if (cause instanceof AssertionError) {
        return false; // Real failure — do not retry
    }

    int count = retryCounts
        .computeIfAbsent(getTestKey(result), k -> new AtomicInteger(0))
        .get();

    if (count < maxRetryCount && isRetryableException(cause)) {
        retryCounts.get(getTestKey(result)).incrementAndGet();
        return true;
    }
    return false;
}
```

## 3. Contract vs Business Test Split
Contract tests verify API format — safe for shared CI environments.
Business tests verify financial logic — require controlled data, run locally.

```java

// CONTRACT — runs in CI on every push
@Test(groups = {"api", "contract", "authenticated"})
public void testGetAccountBalance_Contract() {
    BigDecimal balance = accountApi.getAccountBalance(testAccountId);
    assertThat("Balance endpoint should return a value", balance, notNullValue());
}

// BUSINESS — excluded from CI via -DexcludedGroups=business
@Test(groups = {"api", "business", "authenticated"})
public void testGetAccountBalance_BusinessLogic() {
    BigDecimal balance = accountApi.getAccountBalance(testAccountId);
    assertThat("Balance should be non-negative", balance,
        greaterThanOrEqualTo(BigDecimal.ZERO));
}

```

## 4. Hybrid Test Pattern
API for fast, stable setup. UI only for final user-facing verification.

```java

@Test(groups = {"e2e", "hybrid", "business"})
public void shouldOpenAccountViaAPI_andVerifyInUI() {

    // 1. Authenticate both API and browser sessions simultaneously
    hybridLogin(getTestUsername(), getTestPassword());

    // 2. Create account via API - faster and more stable than UI form 
    OpenAccountRequest request = new OpenAccountRequest(
            "SAVINGS", getTestAccountId(), getTestCustomerId());
    Response response = accountApi.createAccount(request);

    // 3. Map raw JSON response to typed DTO
    OpenAccountDTO accountDTO = OpenAccountApiMapper.from(response);

    // 4. Navigate browser directly to new account URL 
    getDriver().get(getParaBankBaseUrl() + "/activity.htm?id=" + accountDTO.getNewAccountNumber());

    // 5. Verify UI reflects what the API created 
    AccountDetailsPage detailsPage = new AccountDetailsPage(getDriver());

    detailsPage.verifyOpenAccount(accountDTO, "SAVINGS");
}
```
