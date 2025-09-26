# Laravel Project Structure & Conventions

This document explains how our Laravel projects are structured, including **actions, DTOs, exceptions, services, interfaces, and pipelines**, with examples.

---

## Folder Structure

```
app/
 ├─ Contracts/                    # All interfaces
 │   └─ ActionInterface.php
 │
 ├─ Actions/
 │   ├─ BaseAction.php
 │   ├─ SaleOrderActions/          # Subfolder for SaleOrder related actions
 │   │   ├─ CreateSaleOrderAction.php
 │   │   ├─ UpdateSaleOrderAction.php
 │   │   └─ CancelSaleOrderAction.php
 │   └─ CustomerActions/
 │       ├─ CreateCustomerAction.php
 │       └─ UpdateCustomerAction.php
 │
 ├─ Data/
 │   ├─ BaseData.php
 │   ├─ SaleOrderData/             # Subfolder for SaleOrder related DTOs
 │   │   ├─ CreateSaleOrderData.php
 │   │   └─ UpdateSaleOrderData.php
 │   └─ CustomerData/
 │       ├─ CreateCustomerData.php
 │       └─ UpdateCustomerData.php
 │
 ├─ Exceptions/
 │   ├─ DomainException.php
 │   └─ SaleOrderCancelledException.php
 │
 ├─ Services/
 │   ├─ PaymentGatewayService.php
 │   └─ ShippingService.php
 │
 └─ Traits/
     └─ ActionPipeline.php
```

---

## 1. Interfaces

- Location: `Contracts/`  
- Naming: end with `Interface`  
- Example:

```php
namespace App\Contracts;

interface ActionInterface
{
    public function execute(): mixed;
}
```

---

## 2. Actions

- Location: `Actions/`  
- **BaseAction** provides:  
  - `execute()` → runs inside a DB transaction by default  
  - `withoutTransaction()` → runs without transaction, useful for pipelines  
  - Shared functionality like pipeline support  

### BaseAction Example

```php
namespace App\Actions;

use Illuminate\Support\Facades\DB;
use App\Contracts\ActionInterface;
use App\Traits\ActionPipeline;

abstract class BaseAction implements ActionInterface
{
    use ActionPipeline;

    public function execute(): mixed
    {
        return DB::transaction(function () {
            return $this->handle();
        });
    }

    public function withoutTransaction(): mixed
    {
        return $this->handle();
    }

    abstract protected function handle(): mixed;
}
```

### Child Action Example

```php
namespace App\Actions\SaleOrderActions;

use App\Actions\BaseAction;
use App\Data\SaleOrderData\CreateSaleOrderData;
use App\Models\Order;

class CreateSaleOrderAction extends BaseAction
{
    public function __construct(private CreateSaleOrderData $data) {}

    protected function handle(): mixed
    {
        $order = Order::create([
            'customer_id' => $this->data->customer->id,
            'total' => $this->data->total,
        ]);

        return $order;
    }
}
```

---

## 3. Pipelines

- Actions can be chained using `pipe()` from **ActionPipeline trait**.  

```php
$order = app(ValidateSaleOrderAction::class)
    ->withoutTransaction()
    ->pipe(CreateSaleOrderAction::class)
    ->pipe(SendOrderConfirmationAction::class);
```

- Each action gets the **result of the previous action**.  
- Ensures **modular, reusable workflows**.

---

## 4. DTOs (Data Transfer Objects)

- Location: `Data/`  
- **BaseData** provides common functionality (e.g., to array conversion).  
- Feature-specific DTOs live in **subfolders**, e.g., `SaleOrderData`.  

### BaseDTO Example

```php
namespace App\Data;

use Spatie\DataTransferObject\DataTransferObject;

abstract class BaseData extends DataTransferObject
{
    public function toDbArray(): array
    {
        return (array) $this;
    }
}
```

### Child DTO Example

```php
namespace App\Data\SaleOrderData;

use App\Data\BaseData;
use App\Models\Customer;

class CreateSaleOrderData extends BaseData
{
    public function __construct(
        public Customer $customer,
        public array $products,
        public float $total
    ) {}
}
```

- Use DTOs for actions with **structured/multiple inputs**.  
- Skip DTOs for **single-model actions**.

---

## 5. Exceptions

- Location: `Exceptions/`  
- Custom exceptions always end with `Exception`.  
- Optional base exception: `DomainException`.

### Example

```php
if ($saleOrder->isCancelled()) {
    throw new SaleOrderCancelledException();
}
```

**Catching Exceptions:**

```php
try {
    $action->execute();
} catch (DomainException $e) {
    // Handle error in UI or API
}
```

---

## 6. Services

- Location: `Services/`  
- Purpose: Third-party API integrations  
- Keep services **stateless**  
- Inject services via constructor in actions  

---

## 7. Naming Conventions

| Type       | Suffix Example                     |
|------------|----------------------------------|
| Enum       | `StatusEnum`                     |
| Interface  | `ReportInterface`                |
| Action     | `CreateInvoiceAction`            |
| Service    | `PaymentGatewayService`          |
| Observer   | `OrderObserver`                  |
| Policy     | `UserPolicy`                     |
| Controller | `OrderController`                |
| Rule       | `UniqueEmailRule`                |
| Settings   | `GeneralSettings`                |
| State      | `OrderPendingState`              |
| DTO        | `OrderData`                      |
| Exception  | `SaleOrderCancelledException`    |
| Test       | `OrderTest`                      |

> **Note:** In our project, we distinguish between the words **state** and **status**. We will **always use "state"** in models, actions, and DTOs.

---

## 8. Testing Guidelines

- Test **actions in isolation**.  
- Use **Laravel container** for instantiation.  
- Mock services injected in constructors.  
- Use DTOs for consistent input and Result DTOs for output.  
- Test **pipelines** to ensure chaining works correctly.

---

## 9. General Notes

- Keep **logic in Actions/Services**, not in controllers.  
- Always pass **models or DTOs**, not raw IDs.  
- Use **custom exceptions** for domain errors.  
- Default `execute()` uses **DB transaction**; `withoutTransaction()` for pipelines.  
- Pipelines help with **modular and reusable workflows**.

---

✅ This setup ensures a **clean, testable, and scalable Laravel project**, even as the project grows.

