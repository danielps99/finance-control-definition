# Domain Model (`model/`) Architecture & Guidelines

This document defines the architectural patterns, layer responsibilities, persistence flow, and concrete domain examples for the `br.com.bdws.financecontrol.model` package in the **Finance Control System MVP**.

---

## 1. Role & Architectural Boundaries

`model/` contains **pure Java domain objects and aggregates** that encapsulate complex business rules, dynamic calculations, and domain invariants. They are explicitly distinct from database entities.

### Layer Comparison Matrix

| Layer | Package | Responsibility | Annotations | Database Access |
| :--- | :--- | :--- | :--- | :--- |
| **API** | `dto/` | Request/Response HTTP payloads | `@NotNull`, `@Valid` | ❌ No |
| **Persistence** | `entity/` | PostgreSQL table mapping (JPA) | `@Entity`, `@Table`, `@Audited` | ✅ ORM mapped |
| **Domain** | `model/` | **Pure business logic & aggregates** | Plain Java (No ORM/CDI) | ❌ No |
| **Data Access**| `repository/`| Panache queries & persistence | `@ApplicationScoped` | ✅ Direct HQL/SQL |
| **Application**| `service/` | **Transactions & Orchestration** | `@ApplicationScoped`, `@Transactional` | ⚡ Via Repositories |

---

## 2. Architectural Flow & Persistence Responsibility

> **Core Invariants:**
> 1. **Persistence:** A `Model` **NEVER** saves itself or interacts with database repositories. The **`Service`** opens the `@Transactional` boundary and invokes `Repository.persist(...)`.
> 2. **Logging:** A `Model` **NEVER** prints or emits logs (no `Logger` or `System.out`). Operation/transaction logging belongs in the `Service` layer, while models signal business rule failures by throwing domain exceptions.

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Resource as PayableGroupResource
    participant Service as PayableGroupService (@Transactional)
    participant Model as InstallmentScheduleModel
    participant Repos as Panache Repositories
    participant DB as PostgreSQL

    Client->>Resource: POST /api/v1/workspaces/{id}/payable-groups (DTO)
    Resource->>Service: createInstallmentGroup(workspaceId, dto)
    Note over Service: Starts @Transactional boundary
    Service->>Model: new InstallmentScheduleModel(description, total, count, dueDate)
    Note over Model: Pure Domain Logic:<br/>Calculates installment dates,<br/>rounds cents, constructs Entities
    Model-->>Service: Returns Aggregate (Parent Entity + List<Child Entity>)
    Service->>Repos: groupRepo.persist(group); payableRepo.persist(installments)
    Repos->>DB: INSERT into payable_groups, payables
    Note over Service: Commits transaction atomically
    Service-->>Resource: Return Group Entity
    Resource-->>Client: 201 Created (Response DTO)
```

---

## 3. End-to-End Implementation Blueprint

### 1. Domain Model (`InstallmentScheduleModel.java`) — *Pure Java*
```java
package br.com.bdws.financecontrol.model;

import br.com.bdws.financecontrol.entity.PayableEntity;
import br.com.bdws.financecontrol.entity.PayableGroupEntity;
import br.com.bdws.financecontrol.vo.Money;
import java.time.LocalDate;
import java.util.*;

public class InstallmentScheduleModel {
    private final PayableGroupEntity groupEntity;
    private final List<PayableEntity> installmentEntities = new ArrayList<>();

    public InstallmentScheduleModel(UUID workspaceId, String name, Money totalAmount, int count, LocalDate firstDueDate) {
        this.groupEntity = new PayableGroupEntity(workspaceId, name, totalAmount.asBigDecimal(), count);

        Money baseAmount = totalAmount.divide(count);
        Money remainder = totalAmount.subtract(baseAmount.multiply(count));

        for (int i = 1; i <= count; i++) {
            Money amount = (i == 1) ? baseAmount.add(remainder) : baseAmount; // Remainder to 1st installment
            PayableEntity item = new PayableEntity(workspaceId, name + " (" + i + "/" + count + ")", amount.asBigDecimal(), firstDueDate.plusMonths(i - 1), i);
            installmentEntities.add(item);
        }
    }

    public PayableGroupEntity getGroupEntity() { return groupEntity; }
    public List<PayableEntity> getInstallmentEntities() { return List.copyOf(installmentEntities); }
}
```

### 2. Application Service (`PayableGroupService.java`) — *Orchestrator*
```java
package br.com.bdws.financecontrol.service;

import br.com.bdws.financecontrol.dto.CreatePayableGroupDTO;
import br.com.bdws.financecontrol.entity.*;
import br.com.bdws.financecontrol.model.InstallmentScheduleModel;
import br.com.bdws.financecontrol.repository.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;

@ApplicationScoped
public class PayableGroupService {
    @Inject PayableGroupRepository groupRepository;
    @Inject PayableRepository payableRepository;

    @Transactional
    public PayableGroupEntity createInstallmentGroup(UUID workspaceId, CreatePayableGroupDTO dto) {
        // 1. Delegate business calculation & entity building to pure Model
        InstallmentScheduleModel model = new InstallmentScheduleModel(workspaceId, dto.description(), dto.totalAmount(), dto.count(), dto.firstDueDate());

        // 2. Persist parent and child entities via Repositories
        groupRepository.persist(model.getGroupEntity());
        model.getInstallmentEntities().forEach(p -> p.setPayableGroupId(model.getGroupEntity().getId()));
        payableRepository.persist(model.getInstallmentEntities());

        return model.getGroupEntity();
    }
}
```

---

## 4. Key Domain Model Patterns in the System

### A. Cash Flow Projection Engine (`CashFlowReportModel`)
*Merges historical realized ledger data (`account_movements`) with future projections (`payables`/`receivables`).*
```java
public record CashFlowReportModel(DateRange range, CashFlowMode mode, List<CashFlowDaySummary> dailySummaries) {
    public Money calculateTotalInflows() {
        return dailySummaries.stream().map(CashFlowDaySummary::totalInflow).reduce(Money.ZERO, Money::add);
    }
    public Money calculateProjectedClosingBalance() {
        return dailySummaries.isEmpty() ? Money.ZERO : dailySummaries.get(dailySummaries.size() - 1).closingBalance();
    }
}
```

### B. Transaction Reversal (`ReversalOperation`)
*Enforces immutable ledger rule ([reversal-logic.md](file:///home/developer/code/danielps99/finance-control-definition/reversal-logic.md)) by creating compensating movements.*
```java
public class ReversalOperation {
    public AccountMovementEntity buildCompensatingMovement(AccountMovementEntity original, String reason) {
        MovementType opposite = original.getType() == MovementType.DEBIT ? MovementType.CREDIT : MovementType.DEBIT;
        return new AccountMovementEntity(original.getAccountId(), original.getAmount(), opposite, "Reversal: " + reason, original.getId());
    }
    public PayableStatus recalculateStatus(Money paidAmount, Money totalAmount, LocalDate dueDate) {
        if (paidAmount.isZero()) return dueDate.isBefore(LocalDate.now()) ? PayableStatus.OVERDUE : PayableStatus.PENDING;
        return paidAmount.isLessThan(totalAmount) ? PayableStatus.PARTIALLY_PAID : PayableStatus.PAID;
    }
}
```

### C. Credit Card Billing Cycle (`CreditCardBillingCycle`)
*Determines dynamic statement cutoff dates based on closing day and due day ([README.md](file:///home/developer/code/danielps99/finance-control-definition/README.md#L51)).*
```java
public record CreditCardBillingCycle(UUID cardId, int closingDay, int dueDay) {
    public LocalDate calculateClosingDate(LocalDate txDate) {
        return txDate.getDayOfMonth() <= closingDay ? txDate.withDayOfMonth(closingDay) : txDate.plusMonths(1).withDayOfMonth(closingDay);
    }
    public LocalDate calculateDueDate(LocalDate closingDate) {
        return dueDay > closingDay ? closingDate.withDayOfMonth(dueDay) : closingDate.plusMonths(1).withDayOfMonth(dueDay);
    }
}
```

### D. Value Object Implementation Example (`Money.java`)
*Immutable domain Value Object representing monetary values with scale 2 precision and HALF_EVEN rounding.*
```java
package br.com.bdws.financecontrol.vo;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public final class Money implements Comparable<Money> {
    public static final Money ZERO = new Money(BigDecimal.ZERO);

    private final BigDecimal amount;

    public Money(BigDecimal amount) {
        Objects.requireNonNull(amount, "Amount cannot be null");
        this.amount = amount.setScale(2, RoundingMode.HALF_EVEN);
    }

    public static Money of(double val) {
        return new Money(BigDecimal.valueOf(val));
    }

    public Money add(Money other) {
        Objects.requireNonNull(other, "Money operand cannot be null");
        return new Money(this.amount.add(other.amount));
    }

    public Money subtract(Money other) {
        Objects.requireNonNull(other, "Money operand cannot be null");
        return new Money(this.amount.subtract(other.amount));
    }

    public Money multiply(int factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)));
    }

    public Money divide(int divisor) {
        if (divisor == 0) throw new ArithmeticException("Division by zero");
        return new Money(this.amount.divide(BigDecimal.valueOf(divisor), 2, RoundingMode.HALF_EVEN));
    }

    public boolean isZero() { return amount.compareTo(BigDecimal.ZERO) == 0; }
    public boolean isLessThan(Money other) { 
        Objects.requireNonNull(other, "Comparison operand cannot be null");
        return amount.compareTo(other.amount) < 0; 
    }
    public BigDecimal asBigDecimal() { return amount; }

    @Override
    public int compareTo(Money o) { 
        Objects.requireNonNull(o, "Comparison operand cannot be null");
        return this.amount.compareTo(o.amount); 
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.equals(money.amount);
    }

    @Override
    public int hashCode() { return Objects.hash(amount); }

    @Override
    public String toString() { return "$" + amount.toPlainString(); }
}
```

---

## 5. Summary Rules for AI & Developers

1. **Keep `model/` Pure**: No `@Inject`, `@Transactional`, `@Entity`, or `PanacheRepository` references in `model/`.
2. **`Service` Persistence**: `Service` methods annotated with `@Transactional` invoke `Repository.persist(...)`.
3. **Use Java Records for Read-Only Models/VOs**: Prefer `record` or immutable classes for domain models (`CashFlowReportModel`, `CreditCardBillingCycle`, `DashboardSnapshot`, `Money`).
4. **Fast Unit Tests**: Test `model/` classes using standard JUnit 5 without booting Quarkus or a database container.
5. **No Logging Side-Effects**: Models must NOT emit logs (`Logger` or `System.out`). Models throw domain exceptions for invalid state; `Service` handles operation logging.
