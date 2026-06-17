---
layout: post
title: "Implementing Optimistic and Pessimistic Locking in Spring Boot and Spring Data JPA"
date: 2023-03-31
categories: java spring-boot
---

The source code of this tutorial is available [over on GitHub](https://github.com/csdavidg/dblockingpoc).

---

Let's suppose there is a bank account and a microservice to handle withdrawals and deposits. Since this service is going to be horizontally scaled up/down depending on the load, it could have several instances trying to update the account balance either with a withdrawal or a deposit.

For these two use cases, when a deposit is done the service must ensure that only one instance updates the balance account at a time to avoid lost updates. However, withdrawals must only be executed if the account has enough money and there are no ongoing deposits. If there is an ongoing deposit, the application must throw an exception.

To ensure data integrity and prevent conflicts during account balance updates, the service uses a combination of optimistic and pessimistic locking, as described below.

For **deposits**, the service uses **pessimistic locking** to lock the row, allowing only one instance at a time to modify the account balance. Other instances attempting to modify the same row will be blocked until the lock is released.

For **withdrawals**, **optimistic locking** is used. If the account is currently locked due to a deposit or withdrawal in progress, the application will return an error message, and the user must try the withdrawal again later.

The microservice will be scaled horizontally into a Kubernetes cluster having four instances to provide the right conditions to demonstrate the behavior described before. The following diagram depicts the Kubernetes cluster.

![High-level architecture diagram](/assets/images/DB_LOCKING_POC.jpg)
*High-level architecture diagram*

In order to run several requests at the same time I created a JMeter test plan that will send 14 requests at the same time going through a CSV file that provides the data for each of them. This will be helpful to visualize the behavior of the service and the results on each scenario.

> 📷 **Image placeholder:** JMeter test plan configuration
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/jmeter-config.png`, then replace this block with `![JMeter test plan configuration](/assets/images/jmeter-config.png)`)*

The JMeter test plan and the CSV file can be found under the `JMeter` folder in the repository.

## First test — without any locking approach

This is the content of the CSV file to test the first scenario. It should update the account balance, with an initial value of 0, and at the end of the test it should be **320**.

```csv
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
```

After the test plan was completed successfully, the account balance is **40**. Why? This is a concurrency error named **"lost update"**, which occurs when two or more transactions attempt to update the same data simultaneously, and one of them overwrites the changes made by the other.

In this case, several instances are querying the account balance from the database at the same time and consequently several instances are adding 20 to the same value. This behavior can be identified by checking the application logs.

> 📷 **Image placeholder:** Application logs of two different instances
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/logs-no-locking.png`, then replace this block with `![Application logs](/assets/images/logs-no-locking.png)`)*

The JMeter results show that multiple requests are using the same data.

> 📷 **Image placeholder:** JMeter results — requests table
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/jmeter-results-1.png`)*

> 📷 **Image placeholder:** JMeter results — summary
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/jmeter-results-2.png`)*

## Second test — with optimistic and pessimistic locking

In order to use optimistic and pessimistic locking as described above, it is necessary to run this logic within a transaction scope. The Spring framework provides the `@Transactional` annotation which manages the transaction using AOP and commits or rolls back the transaction depending on the outcome.

More information about [Spring transaction management and the `@Transactional` annotation](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction).

**BankAccountService.java**

```java
@Transactional
public AccountTransaction accountTransaction(AccountTransaction transaction) {

    BankAccount e;
    double newBalance;

    switch (transaction.type()) {
        case DEBIT -> {
            e = repository.findByAccountNumberAndName(transaction.accountNumber(), transaction.name())
                    .orElseThrow();
            newBalance = e.getBalance() - transaction.amount();
            if (newBalance < 0) {
                throw new IllegalArgumentException("Insufficient funds");
            }
        }
        case CREDIT -> {
            e = repository.findByAccountNumber(transaction.accountNumber())
                    .orElseThrow();
            System.out.println("Old value " + e.getBalance());
            newBalance = e.getBalance() + transaction.amount();
        }
        default -> throw new IllegalArgumentException("Invalid transaction type");
    }
    System.out.println("New value " + newBalance);
    e.setBalance(newBalance);
    repository.save(e);
    return new AccountTransaction(transaction.accountNumber(), transaction.name(), transaction.type(), newBalance);
}
```

**BankAccountRepository.java**

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<BankAccount> findByAccountNumber(String accountNumber);

@Lock(LockModeType.OPTIMISTIC)
Optional<BankAccount> findByAccountNumberAndName(String accountNumber, String name);
```

This is the content of the CSV file to test the second scenario. It should update the account balance with deposits and withdrawals; the initial account balance will be 100 (if a DEBIT is executed first, it will avoid the `Insufficient funds` exception).

```csv
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,DEBIT,20
90898329,Julio,DEBIT,20
90898329,Julio,CREDIT,20
90898329,Julio,DEBIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,CREDIT,20
90898329,Julio,DEBIT,20
90898329,Julio,DEBIT,20
90898329,Julio,DEBIT,20
```

After the test plan was executed, **14 CREDITS (Deposits)** and **2 DEBITS (Withdrawals)** were saved successfully, and the account balance is **340**, which is the expected value using pessimistic locking for deposits and optimistic locking for withdrawals.

```
(Account balance + successful deposits) – successful withdrawals
(100 + 280) – 40 = 340
```

This time the JMeter test plan shows that some DEBIT requests were not processed further.

> 📷 **Image placeholder:** JMeter test plan — View results tree
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/jmeter-results-tree.png`, then replace this block with `![JMeter results tree](/assets/images/jmeter-results-tree.png)`)*

The withdrawals use optimistic locking to ensure that the record's field version is tracked when updating the data. This ensures that the most up-to-date value is used, and in case any other instance has updated the data, an exception is thrown.

> 📷 **Image placeholder:** Optimistic locking exception thrown
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/optimistic-lock-exception.png`, then replace this block with `![Optimistic locking exception](/assets/images/optimistic-lock-exception.png)`)*

The application logs show the behavior of the service using the locking techniques to prevent conflicts. It is worth noting the old and new values of the account balance and when the exception is thrown.

> 📷 **Image placeholder:** Application logs from two instances
> *(Save this screenshot from the LinkedIn article and place it at `assets/images/logs-with-locking.png`, then replace this block with `![Application logs with locking](/assets/images/logs-with-locking.png)`)*

## Conclusions

In conclusion, the use of pessimistic and optimistic locking in Spring is fairly straightforward and helps to avoid the lost update problem. However, both approaches have trade-offs that must be considered and managed carefully depending on the context and the application needs.

### Pessimistic locking

**Advantages:**
- Ensures data consistency by preventing concurrent transactions from modifying data simultaneously.
- Straightforward to implement.
- Reduced risk of conflicts since only one transaction can access and modify the data at a time.
- Supported by most relational database systems.

**Disadvantages:**
- Reduced concurrency by blocking concurrent transactions.
- Increased contention since transactions that require access to the data must wait until the lock is released.
- Potential deadlocks where several transactions are waiting for lock release.
- Higher resource utilization since locks must be held for the duration of the transaction.

### Optimistic locking

**Advantages:**
- Reduced contention by allowing multiple transactions to access the data concurrently (other transactions can read the data but not update it if another transaction has already done so).
- Improved scalability since it does not block concurrent access to the data.
- Reduced overhead since locks don't need to be acquired and released explicitly.

**Disadvantages:**
- Risk of conflicts since optimistic locking relies on the assumption that conflicts between transactions are rare.
- Additional complexity due to the extra code required to handle rollbacks or retries.
- Limited support — some databases do not support or have limited support for optimistic locking.
