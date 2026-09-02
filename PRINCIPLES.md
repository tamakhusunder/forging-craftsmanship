# Foundational Principles

These principles help keep software understandable, changeable, and aligned with
the knowledge it represents. They are guidelines for managing coupling and
duplication, not rigid rules to apply without context.

## 1. DRY: Don't Repeat Yourself

DRY means that each piece of knowledge or intent should have one authoritative
representation in a system. The goal is to reduce duplicated decisions and
business rules, not merely to eliminate similar-looking lines of code. Duplication
can be acceptable when two things only happen to look alike but represent
different concepts.

### Before

The validation rule is duplicated in two places. A change to the password policy
can update one path while leaving the other incorrect.

```ts
function register(password: string): void {
  if (password.length < 12) {
    throw new Error('Password must be at least 12 characters');
  }

  // Create the account...
}

function changePassword(password: string): void {
  if (password.length < 12) {
    throw new Error('Password must be at least 12 characters');
  }

  // Save the new password...
}
```

### After

The password policy has one authoritative implementation, so both operations use
the same knowledge.

```ts
const MIN_PASSWORD_LENGTH = 12;

function validatePassword(password: string): void {
  if (password.length < MIN_PASSWORD_LENGTH) {
    throw new Error(`Password must be at least ${MIN_PASSWORD_LENGTH} characters`);
  }
}

function register(password: string): void {
  validatePassword(password);
  // Create the account...
}

function changePassword(password: string): void {
  validatePassword(password);
  // Save the new password...
}
```

## SOLID

SOLID is a set of five object-oriented design principles that encourage focused
responsibilities, safe extension, substitutable abstractions, small interfaces,
and dependencies on stable abstractions.

## 2. Single Responsibility Principle

A module or class should have one responsibility and therefore one reason to
change.

A class should focus on one cohesive concern rather than handling unrelated responsibilities.

### Before

`UserService` handles user creation, password hashing, database persistence, and email notification.

```typescript
class UserService {
  async createUser(email: string, password: string) {
    // Hash password
    const hashedPassword = await this.hashPassword(password);

    // Save user
    const user = await this.saveToDatabase({
      email,
      password: hashedPassword,
    });

    // Send email
    await this.sendWelcomeEmail(email);

    return user;
  }

  private async hashPassword(password: string) {
    // Password hashing logic
    return `hashed-${password}`;
  }

  private async saveToDatabase(user: object) {
    // Database logic
    return user;
  }

  private async sendWelcomeEmail(email: string) {
    // Email logic
    console.log(`Welcome email sent to ${email}`);
  }
}
```

This class has several reasons to change:

- Password hashing changes
- Database implementation changes
- Email provider changes
- User creation logic changes

### After

Separate responsibilities into focused classes:

```ts
class PasswordHasher {
  async hash(password: string): Promise<string> {
    return `hashed-${password}`;
  }
}

class UserRepository {
  async save(user: { email: string; password: string }) {
    // Database logic
    return user;
  }
}

class EmailService {
  async sendWelcomeEmail(email: string): Promise<void> {
    console.log(`Welcome email sent to ${email}`);
  }
}

class UserService {
  constructor(
    private readonly passwordHasher: PasswordHasher,
    private readonly userRepository: UserRepository,
    private readonly emailService: EmailService
  ) {}

  async createUser(email: string, password: string) {
    const hashedPassword = await this.passwordHasher.hash(password);

    const user = await this.userRepository.save({
      email,
      password: hashedPassword,
    });

    await this.emailService.sendWelcomeEmail(email);

    return user;
  }
}
```

Each class now has a focused responsibility.

## Open-Closed Principle

Software entities should be open for extension but closed for modification. New
behavior should usually be added through new implementations or composed rules,
rather than repeatedly editing stable, existing logic.

### Before

Adding another payment method requires modifying the existing class:

```typescript
class PaymentService {
  pay(method: string, amount: number) {
    if (method === 'card') {
      console.log(`Paid ${amount} using card`);
    } else if (method === 'paypal') {
      console.log(`Paid ${amount} using PayPal`);
    } else if (method === 'esewa') {
      console.log(`Paid ${amount} using eSewa`);
    }
  }
}
```

Every new payment method requires changing `PaymentService`.

### After

Use an abstraction that allows new payment methods to be added without modifying the payment service:

```typescript
interface PaymentMethod {
  pay(amount: number): void;
}

class CardPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Paid ${amount} using card`);
  }
}

class PaypalPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Paid ${amount} using PayPal`);
  }
}

class EsewaPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Paid ${amount} using eSewa`);
  }
}

class PaymentService {
  pay(paymentMethod: PaymentMethod, amount: number): void {
    paymentMethod.pay(amount);
  }
}
```

Now a new payment method can be added:

```typescript
class KhaltiPayment implements PaymentMethod {
  pay(amount: number): void {
    console.log(`Paid ${amount} using Khalti`);
  }
}
```

`PaymentService` does not need to change.

## Liskov Substitution Principle

Subtypes must be usable wherever their base type is expected without breaking the
base type's behavioral guarantees. Inheritance or implementation should preserve
the abstraction's valid inputs, outputs, and expectations.

If `B` is a subtype of `A`, code using `A` should work correctly when given `B`.

### Before

A `Penguin` is technically a bird, but it cannot fly.

```typescript
class Bird {
  fly(): void {
    console.log('Flying...');
  }
}

class Eagle extends Bird {
  fly(): void {
    console.log('Eagle is flying');
  }
}

class Penguin extends Bird {
  fly(): void {
    throw new Error('Penguins cannot fly');
  }
}

function makeBirdFly(bird: Bird): void {
  bird.fly();
}

const bird: Bird = new Penguin();

makeBirdFly(bird); // Runtime error
```

`Penguin` cannot safely substitute `Bird` because the parent abstraction promises flying behavior.

### After

Separate the concepts:

```typescript
class Bird {
  eat(): void {
    console.log('Eating...');
  }
}

interface FlyingBird {
  fly(): void;
}

class Eagle extends Bird implements FlyingBird {
  fly(): void {
    console.log('Eagle is flying');
  }
}

class Penguin extends Bird {
  swim(): void {
    console.log('Penguin is swimming');
  }
}

function makeBirdFly(bird: FlyingBird): void {
  bird.fly();
}

makeBirdFly(new Eagle());
```

Now only birds that actually support flying implement `FlyingBird`.

The subtype satisfies the contract expected by the caller.

## Interface Segregation Principle

Clients should not be forced to depend on methods they do not use. Prefer several
small, role-specific interfaces over one large interface that makes every
implementation provide irrelevant or fake operations.

Prefer several small, focused interfaces over one large interface.

### Before

A single interface forces every employee to implement unrelated methods:

```typescript
interface Employee {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Robot implements Employee {
  work(): void {
    console.log('Robot working');
  }

  eat(): void {
    throw new Error('Robot does not eat');
  }

  sleep(): void {
    throw new Error('Robot does not sleep');
  }
}
```

The robot is forced to implement methods that do not make sense.

### After

Split the large interface into smaller interfaces:

```typescript
interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

interface Sleepable {
  sleep(): void;
}

class Human implements Workable, Eatable, Sleepable {
  work(): void {
    console.log('Human working');
  }

  eat(): void {
    console.log('Human eating');
  }

  sleep(): void {
    console.log('Human sleeping');
  }
}

class Robot implements Workable {
  work(): void {
    console.log('Robot working');
  }
}
```

Now each class implements only the capabilities it actually needs.

## Dependency Inversion Principle

High-level policy should not depend directly on low-level details. Both should
depend on abstractions, and those abstractions should be owned by the policy that
needs them. Details then become replaceable, such as in tests or when changing
infrastructure.

### Before

`UserService` directly depends on `MySQLDatabase`:

```typescript
class MySQLDatabase {
  saveUser(email: string): void {
    console.log(`Saving ${email} to MySQL`);
  }
}

class UserService {
  private database = new MySQLDatabase();

  createUser(email: string): void {
    this.database.saveUser(email);
  }
}
```

The high-level business logic is tightly coupled to MySQL.

Changing to PostgreSQL would require modifying `UserService`.

### After

Depend on an abstraction:

```typescript
interface UserRepository {
  saveUser(email: string): void;
}

class MySQLUserRepository implements UserRepository {
  saveUser(email: string): void {
    console.log(`Saving ${email} to MySQL`);
  }
}

class PostgreSQLUserRepository implements UserRepository {
  saveUser(email: string): void {
    console.log(`Saving ${email} to PostgreSQL`);
  }
}

class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  createUser(email: string): void {
    this.userRepository.saveUser(email);
  }
}
```

Now the service is independent of the database implementation:

```typescript
const mysqlRepository = new MySQLUserRepository();

const userService = new UserService(mysqlRepository);

userService.createUser('user@example.com');
```

We can switch to PostgreSQL without changing `UserService`:

```typescript
const postgresRepository = new PostgreSQLUserRepository();

const userService = new UserService(postgresRepository);

userService.createUser('user@example.com');
```

# Summary

| Principle | Main Idea                                            |
| --------- | ---------------------------------------------------- |
| **DRY**   | Don't duplicate knowledge or business rules          |
| **SRP**   | One class/module should have one responsibility      |
| **OCP**   | Extend behavior without modifying stable code        |
| **LSP**   | Subtypes must safely replace their parent types      |
| **ISP**   | Prefer small, focused interfaces                     |
| **DIP**   | Depend on abstractions, not concrete implementations |

## Practical Mental Model

```text
DRY
 ↓
Avoid duplicated knowledge

SRP
 ↓
Keep responsibilities focused

OCP
 ↓
Add behavior without modifying existing behavior

LSP
 ↓
Make implementations safely substitutable

ISP
 ↓
Keep interfaces small and focused

DIP
 ↓
Keep business logic independent from implementation details
```

These principles are guidelines rather than rigid rules. Applying them blindly can introduce unnecessary abstractions. The goal is to improve **maintainability, flexibility, testability, and clarity** while keeping the design appropriately simple.
