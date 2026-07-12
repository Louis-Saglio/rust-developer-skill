# Example: Compliant Function Following Rust Developer Principles

This file demonstrates a realistic function written according to the principles in `SKILL.md`.

## Scenario

We need a function that validates and processes a customer order for a specific product quantity. The function must:

- Ensure the customer exists and is active.
- Ensure the product exists and is in stock.
- Validate the requested quantity is at least one and does not exceed stock.
- Compute a total price with a simple bulk discount.
- Return an order summary without performing any side effects.

The repository layer provides pure lookup functions that return `Option<T>`. The order processing function stays pure: it does not log, write to a database, or mutate external state.

## Code

```rust
use rust_decimal::Decimal;

// ---------------------------------------------------------------------------
// Domain newtypes. Bare primitives are never used to represent domain ideas.
// ---------------------------------------------------------------------------

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct CustomerId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ProductId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Quantity(u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct UnitPrice(Decimal);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct OrderTotal(Decimal);

// ---------------------------------------------------------------------------
// Domain enums. Invalid states are unrepresentable.
// ---------------------------------------------------------------------------

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CustomerStatus {
    Active,
    Suspended,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ProductAvailability {
    InStock(u32),
    OutOfStock,
}

// ---------------------------------------------------------------------------
// Pure, read-only input data. No repository handles or side effects leak in.
// ---------------------------------------------------------------------------

#[derive(Debug, Clone)]
pub struct Customer {
    pub id: CustomerId,
    pub status: CustomerStatus,
}

#[derive(Debug, Clone)]
pub struct Product {
    pub id: ProductId,
    pub unit_price: UnitPrice,
    pub availability: ProductAvailability,
}

#[derive(Debug, Clone)]
pub struct OrderSummary {
    pub customer_id: CustomerId,
    pub product_id: ProductId,
    pub quantity: Quantity,
    pub total: OrderTotal,
}

// ---------------------------------------------------------------------------
// Custom error type dedicated to this function. It is not reused elsewhere.
// ---------------------------------------------------------------------------

#[derive(Debug, thiserror::Error)]
pub enum ProcessCustomerOrderError {
    #[error("customer {0:?} not found")]
    CustomerNotFound(CustomerId),

    #[error("customer {0:?} is not active")]
    CustomerNotActive(CustomerId),

    #[error("product {0:?} not found")]
    ProductNotFound(ProductId),

    #[error("product {0:?} is out of stock")]
    ProductOutOfStock(ProductId),

    #[error("requested quantity {requested} exceeds available stock {available}")]
    InsufficientStock {
        requested: Quantity,
        available: u32,
    },

    #[error("quantity must be at least one, got {0}")]
    InvalidQuantity(u32),
}

// ---------------------------------------------------------------------------
// The compliant function. Pure, self-contained, explicit errors, no unwrap.
// ---------------------------------------------------------------------------

/// Builds a validated order summary for an active customer and an in-stock product.
///
/// This function is pure: it depends only on its inputs, performs no I/O, and
/// produces no side effects. Callers (e.g. an HTTP handler or CLI command) are
/// responsible for fetching the inputs and persisting the resulting summary.
pub fn process_customer_order(
    customer_id: CustomerId,
    product_id: ProductId,
    requested_quantity: u32,
    find_customer: impl FnOnce(CustomerId) -> Option<Customer>,
    find_product: impl FnOnce(ProductId) -> Option<Product>,
) -> Result<OrderSummary, ProcessCustomerOrderError> {
    // 1. Validate primitive input and convert it into a domain newtype.
    if requested_quantity == 0 {
        return Err(ProcessCustomerOrderError::InvalidQuantity(requested_quantity));
    }
    let quantity = Quantity(requested_quantity);

    // 2. Resolve customer and enforce business rule: only active customers may order.
    let customer = find_customer(customer_id)
        .ok_or(ProcessCustomerOrderError::CustomerNotFound(customer_id))?;

    if customer.status != CustomerStatus::Active {
        return Err(ProcessCustomerOrderError::CustomerNotActive(customer_id));
    }

    // 3. Resolve product and enforce stock rules.
    let product = find_product(product_id)
        .ok_or(ProcessCustomerOrderError::ProductNotFound(product_id))?;

    let available = match product.availability {
        ProductAvailability::InStock(amount) => amount,
        ProductAvailability::OutOfStock => {
            return Err(ProcessCustomerOrderError::ProductOutOfStock(product_id));
        }
    };

    if requested_quantity > available {
        return Err(ProcessCustomerOrderError::InsufficientStock {
            requested: quantity,
            available,
        });
    }

    // 4. Compute the total with a simple bulk discount. Pure arithmetic, no side effects.
    let UnitPrice(price) = product.unit_price;
    let discount = if requested_quantity >= 10 {
        Decimal::new(95, 2) // 5 % discount
    } else {
        Decimal::ONE
    };
    let total = OrderTotal(price * Decimal::from(requested_quantity) * discount);

    // 5. Return the validated summary.
    Ok(OrderSummary {
        customer_id,
        product_id,
        quantity,
        total,
    })
}
```

## Commentary

- **Newtypes:** `CustomerId`, `ProductId`, `Quantity`, `UnitPrice`, and `OrderTotal` wrap primitives so the type system prevents mixing unrelated values.
- **Enums:** `CustomerStatus` and `ProductAvailability` make invalid states unrepresentable. A product cannot simultaneously be in stock and out of stock.
- **Dedicated error type:** `ProcessCustomerOrderError` is local to this function and describes every possible failure mode precisely. No other function returns it.
- **No `unwrap`/`expect`:** The only fallible operation, the customer/product lookup, is handled explicitly with `ok_or` and `?`.
- **Pure function:** `process_customer_order` takes lookup closures instead of repository references, so it has no I/O, no logging, and no mutation of external state. It is trivial to unit test.
- **Self-contained:** All validation, business-rule enforcement, and price computation live in one cohesive function. The reader can understand the whole operation by reading top to bottom.
- **Standard-library-first:** `rust_decimal` is used only because precise decimal arithmetic is genuinely safer than floating point for money. Otherwise the function relies on the standard library.

## Example Unit Test

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn customer(id: u64, status: CustomerStatus) -> Customer {
        Customer {
            id: CustomerId(id),
            status,
        }
    }

    fn product(id: u64, price: &str, stock: u32) -> Product {
        Product {
            id: ProductId(id),
            unit_price: UnitPrice(price.parse().unwrap()), // infallible in test data
            availability: ProductAvailability::InStock(stock),
        }
    }

    #[test]
    fn happy_path_computes_total_without_discount() {
        let summary = process_customer_order(
            CustomerId(1),
            ProductId(10),
            3,
            |_| Some(customer(1, CustomerStatus::Active)),
            |_| Some(product(10, "12.50", 100)),
        )
        .unwrap();

        assert_eq!(summary.customer_id, CustomerId(1));
        assert_eq!(summary.quantity, Quantity(3));
        assert_eq!(summary.total, OrderTotal(Decimal::new(3750, 2)));
    }

    #[test]
    fn quantity_zero_is_rejected() {
        let err = process_customer_order(
            CustomerId(1),
            ProductId(10),
            0,
            |_| Some(customer(1, CustomerStatus::Active)),
            |_| Some(product(10, "12.50", 100)),
        )
        .unwrap_err();

        assert!(matches!(err, ProcessCustomerOrderError::InvalidQuantity(0)));
    }
}
```
