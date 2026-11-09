# Python API Testing — Code Snippets

Full source code is in the private repository.
These snippets demonstrate the architecture of my Pytest suite.

---

## 1. Session Management with Pytest Fixtures (`conftest.py`)

Rather than hardcoding credentials in every test, I used session-scoped fixtures.
The login runs once, extracts the Customer ID, and shares state across all tests —
following the DRY principle and reducing execution time.

```python
@pytest.fixture(scope="session")
def customer_id(api_session):
    """Log in once and retrieve the customer ID for the whole session."""
    response = api_session.get(f"{BASE_URL}/login/{USERNAME}/{PASSWORD}")
    return response.json()["id"]

@pytest.fixture(scope="session")
def account_ids(api_session, customer_id):
    """Fetch and return the first two account IDs."""
    response = api_session.get(f"{BASE_URL}/customers/{customer_id}/accounts")
    accounts = response.json()
    return {"from": accounts[0]["id"], "to": accounts[1]["id"]} 
    
```
    

## 2. Data-Driven Negative Testing with @pytest.mark.parametrize

The same negative test runs automatically against multiple invalid inputs
(zero, negative, and extremely large amounts) without duplicating code.
Note: The assertion expects 200 OK to document a known ParaBank API bug —
the server incorrectly accepts invalid transfer amounts instead of rejecting them.

```python
@pytest.mark.parametrize("amount", [0, -50, 9999999])
def test_invalid_transfers(api_session, account_ids, amount):
    """Negative Test: Confirm ParaBank accepts invalid amounts (documented bug)."""
    url = (
        f"{BASE_URL}/transfer"
        f"?fromAccountId={account_ids['from']}"
        f"&toAccountId={account_ids['to']}"
        f"&amount={amount}"
    )
    response = api_session.post(url)
    assert response.status_code == 200  # Bug: should be 4xx
    
```


## 3. State Verification — Balance Check
A 200 OK response is not enough for a financial transaction.
I verify the actual account state before and after the transfer to confirm
the correct amount was deducted.

```python

def test_transfer_state_verification(api_session, account_ids):
    """Verify that balance decreases by exactly the transferred amount."""
    amount = 50.00
    from_acc = account_ids["from"]

    # 1. Record balance before transfer
    balance_before = float(
        api_session.get(f"{BASE_URL}/accounts/{from_acc}").json()["balance"]
    )

    # 2. Execute transfer
    api_session.post(
        f"{BASE_URL}/transfer"
        f"?fromAccountId={from_acc}"
        f"&toAccountId={account_ids['to']}"
        f"&amount={amount}"
    )

    # 3. Record balance after transfer
    balance_after = float(
        api_session.get(f"{BASE_URL}/accounts/{from_acc}").json()["balance"]
    )

    # 4. State verification — exact amount must be deducted
    assert round(balance_before - balance_after, 2) == amount

```