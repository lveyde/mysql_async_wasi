# mysql_async for WebAssembly

Tokio based asynchronous MySql client library for The Rust Programming Language.
This is a fork from the original [mysql_async](https://github.com/blackbeam/mysql_async) with support for WebAssembly compilation target.
That allows async MySql apps to run inside the [WasmEdge Runtime](https://github.com/WasmEdge/WasmEdge#readme) as a lightweight and secure alternative to natively compiled apps in Linux container.

For more details and usage examples, please see the upstream [mysql_async](https://github.com/blackbeam/mysql-async) source and [this example](https://github.com/WasmEdge/wasmedge-db-examples/tree/main/mysql_async).

Note: We do not yet support SSL / TLS connections to databases in this WebAssembly client.

## Installation

```toml
[dependencies]
mysql_async_wasi = "<desired version>"
```

## Example

```rust
use mysql_async::prelude::*;

#[derive(Debug, PartialEq, Eq, Clone)]
struct Payment {
    customer_id: i32,
    amount: i32,
    account_name: Option<String>,
}

#[tokio::main]
async fn main() -> Result<()> {
    let payments = vec![
        Payment { customer_id: 1, amount: 2, account_name: None },
        Payment { customer_id: 3, amount: 4, account_name: Some("foo".into()) },
        Payment { customer_id: 5, amount: 6, account_name: None },
        Payment { customer_id: 7, amount: 8, account_name: None },
        Payment { customer_id: 9, amount: 10, account_name: Some("bar".into()) },
    ];

    let database_url = /* ... */
    # get_opts();

    let pool = mysql_async::Pool::new(database_url);
    let mut conn = pool.get_conn().await?;

    // Create a temporary table
    r"CREATE TEMPORARY TABLE payment (
        customer_id int not null,
        amount int not null,
        account_name text
    )".ignore(&mut conn).await?;

    // Save payments
    r"INSERT INTO payment (customer_id, amount, account_name)
      VALUES (:customer_id, :amount, :account_name)"
        .with(payments.iter().map(|payment| params! {
            "customer_id" => payment.customer_id,
            "amount" => payment.amount,
            "account_name" => payment.account_name.as_ref(),
        }))
        .batch(&mut conn)
        .await?;

    // Load payments from the database. Type inference will work here.
    let loaded_payments = "SELECT customer_id, amount, account_name FROM payment"
        .with(())
        .map(&mut conn, |(customer_id, amount, account_name)| Payment { customer_id, amount, account_name })
        .await?;

    // Dropped connection will go to the pool
    drop(conn);

    // The Pool must be disconnected explicitly because
    // it's an asynchronous operation.
    pool.disconnect().await?;

    assert_eq!(loaded_payments, payments);

    // the async fn returns Result, so
    Ok(())
}
```

## License

Licensed under either of

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0)
* MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.
