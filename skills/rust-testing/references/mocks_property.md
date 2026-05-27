# Mocks and Property Testing

Use mocks sparingly and property tests for broad input coverage.

## Trait-Based Test Doubles

Design dependencies behind small traits.

```rust
pub trait EmailSender {
    fn send(&self, to: &str, body: &str) -> Result<(), EmailError>;
}

pub struct RecordingSender {
    pub sent: std::cell::RefCell<Vec<String>>,
}

impl EmailSender for RecordingSender {
    fn send(&self, to: &str, _: &str) -> Result<(), EmailError> {
        self.sent.borrow_mut().push(to.to_string());
        Ok(())
    }
}
```

## mockall

Use `mockall` when expectations are clearer than a hand-written fake.

```rust
#[cfg_attr(test, mockall::automock)]
pub trait Repository {
    fn find_user(&self, id: UserId) -> Result<Option<User>, Error>;
}

// In test:
let mut mock = MockRepository::new();
mock.expect_find_user()
    .with(predicate::eq(UserId(1)))
    .times(1)
    .returning(|_| Ok(Some(User::default())));

let service = UserService::new(Box::new(mock));
```

### When to Use Mocks vs Fakes

| Approach | When to use | Trade-off |
|----------|------------|-----------|
| **Hand-written fake** | Simple state, few methods | More code, zero magic |
| **mockall automated mock** | Many methods, complex expectations | Macro magic, brittle to refactors |
| **Real implementation** | No side effects, fast | Best fidelity, no isolation |
| **Real implementation (test container)** | Database, message queue | Slower, more setup, highest fidelity |

## Property Testing

Use `proptest` to assert invariants across many generated inputs.

```rust
proptest::proptest! {
    #[test]
    fn parse_display_roundtrip(id in 0_u64..) {
        let user_id = UserId(id);
        let parsed: UserId = user_id.to_string().parse().unwrap();
        prop_assert_eq!(parsed, user_id);
    }
}
```

### Custom Strategies

```rust
use proptest::prelude::*;

fn arb_email() -> impl Strategy<Value = String> {
    "[a-z]{2,10}@[a-z]{2,10}\\.[a-z]{2,3}"
}

fn arb_user() -> impl Strategy<Value = User> {
    (arb_email(), "[A-Z][a-z]{2,20}", 0_u64..100_000).prop_map(|(email, name, id)| {
        User { id: UserId(id), name, email }
    })
}

proptest! {
    #[test]
    fn user_validation_accepts_valid(user in arb_user()) {
        assert!(validate_user(&user).is_ok());
    }
}
```

## Good Properties to Test

- Parse/display roundtrips
- Serialization/deserialization roundtrips
- Sorting output is ordered and preserves elements
- Ledger entries sum to zero
- Encoders never panic on arbitrary bytes
- Adding element X to a collection increases `.len()` by 1
- Removing an element from a set means `.contains()` returns false
- Pagination: all pages combined equal the full list

## Regression Seeds

When property tests fail, commit the minimized case as a normal unit test.

```rust
#[test]
fn test_regression_20260527_overflow() {
    // Previously failed: custom strategy generated u64::MAX which caused overflow
    let input = u64::MAX;
    assert!(process_amount(input).is_err());
}
```

## Mock Anti-Patterns

```rust
// Bad: mocking everything — tests pass but real behavior is untested
let mock_db = MockDatabase::new();
mock_db.expect_query().returning(|_| Ok(vec![]));
let service = Service::new(mock_db);

// Better: test the real query against a test database
#[sqlx::test(migrations = "./migrations")]
async fn test_query(pool: PgPool) {
    let service = Service::new(pool);
    let results = service.query(&["user_1", "user_2"]).await.unwrap();
    assert_eq!(results.len(), 2);
}
```

- Avoid mocking types you don't own — use a trait boundary instead.
- Don't mock to avoid database setup — use test containers or SQLx test helpers.
- Mock expectations that go unused = stale tests. Run with `strict()` mode.
