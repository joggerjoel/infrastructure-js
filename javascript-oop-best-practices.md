# JavaScript & TypeScript OOP Best Practices

---
**Last Updated**: 2026-01-28  
**Purpose**: Deep dive into proper OOP and JavaScript semantics  
**Audience**: JavaScript/TypeScript developers  
---

## Table of Contents

- [JavaScript Fundamentals](#javascript-fundamentals)
- [OOP in JavaScript](#oop-in-javascript)
- [TypeScript Enhancements](#typescript-enhancements)
- [Common Pitfalls](#common-pitfalls)
- [Best Practices](#best-practices)
- [Design Patterns](#design-patterns)

---

## JavaScript Fundamentals

### The Prototype Chain

JavaScript is **prototype-based**, not class-based (even with ES6 classes).

```javascript
// What you write (ES6 class syntax)
class Animal {
  constructor(name) {
    this.name = name;
  }
  
  speak() {
    console.log(`${this.name} makes a sound`);
  }
}

// What JavaScript actually does (prototype-based)
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  console.log(`${this.name} makes a sound`);
};

// Both create the same prototype chain:
const dog = new Animal('Dog');
// dog.__proto__ === Animal.prototype
// Animal.prototype.__proto__ === Object.prototype
// Object.prototype.__proto__ === null
```

### `this` Binding Rules (Critical!)

```javascript
// Rule 1: Default binding (strict mode: undefined, non-strict: global)
function foo() {
  console.log(this);  // undefined (strict mode)
}
foo();

// Rule 2: Implicit binding (object method)
const obj = {
  name: 'Object',
  greet() {
    console.log(this.name);  // 'Object'
  }
};
obj.greet();

// Rule 3: Explicit binding (call, apply, bind)
function greet() {
  console.log(this.name);
}
const person = { name: 'Alice' };
greet.call(person);  // 'Alice'

// Rule 4: new binding
function Person(name) {
  this.name = name;  // 'this' is the new object
}
const alice = new Person('Alice');

// Rule 5: Arrow functions (lexical this)
const obj2 = {
  name: 'Object2',
  greet: () => {
    console.log(this.name);  // undefined! Arrow functions don't have 'this'
  },
  greetCorrect() {
    setTimeout(() => {
      console.log(this.name);  // 'Object2' - inherits from greetCorrect
    }, 100);
  }
};
```

### Closures & Scope

```javascript
// Closure: Inner function has access to outer function's variables
function createCounter() {
  let count = 0;  // Private variable
  
  return {
    increment() {
      count++;
      return count;
    },
    decrement() {
      count--;
      return count;
    },
    getCount() {
      return count;
    }
  };
}

const counter = createCounter();
counter.increment();  // 1
counter.increment();  // 2
counter.getCount();   // 2
// counter.count is undefined - truly private!

// Common mistake: Closures in loops
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // Prints: 3, 3, 3
}

// Fix 1: Use let (block scope)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // Prints: 0, 1, 2
}

// Fix 2: IIFE (Immediately Invoked Function Expression)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);  // Prints: 0, 1, 2
  })(i);
}
```

---

## OOP in JavaScript

### ES6 Classes (Syntactic Sugar)

```javascript
// Modern class syntax
class Animal {
  // Public field (ES2022)
  species = 'Unknown';
  
  // Private field (ES2022)
  #privateField = 'secret';
  
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  
  // Instance method
  speak() {
    return `${this.name} makes a sound`;
  }
  
  // Getter
  get info() {
    return `${this.name} is ${this.age} years old`;
  }
  
  // Setter
  set info(value) {
    const [name, age] = value.split(',');
    this.name = name;
    this.age = parseInt(age);
  }
  
  // Private method (ES2022)
  #privateMethod() {
    return this.#privateField;
  }
  
  // Static method
  static createBaby(name) {
    return new Animal(name, 0);
  }
  
  // Static field
  static kingdom = 'Animalia';
}

// Inheritance
class Dog extends Animal {
  constructor(name, age, breed) {
    super(name, age);  // MUST call super() first
    this.breed = breed;
  }
  
  // Override method
  speak() {
    return `${this.name} barks`;
  }
  
  // Call parent method
  speakLikeAnimal() {
    return super.speak();
  }
}

const dog = new Dog('Rex', 3, 'Labrador');
dog.speak();  // 'Rex barks'
dog.speakLikeAnimal();  // 'Rex makes a sound'
```

### Composition over Inheritance

```javascript
// ❌ Bad: Deep inheritance hierarchy
class Animal {}
class Mammal extends Animal {}
class Carnivore extends Mammal {}
class Dog extends Carnivore {}  // Fragile, hard to maintain

// ✅ Good: Composition
const canEat = (state) => ({
  eat(food) {
    console.log(`${state.name} eats ${food}`);
  }
});

const canWalk = (state) => ({
  walk() {
    console.log(`${state.name} walks`);
  }
});

const canSwim = (state) => ({
  swim() {
    console.log(`${state.name} swims`);
  }
});

// Compose behaviors
const createDog = (name) => {
  const state = { name };
  return Object.assign(
    {},
    canEat(state),
    canWalk(state),
    canSwim(state)
  );
};

const createFish = (name) => {
  const state = { name };
  return Object.assign(
    {},
    canEat(state),
    canSwim(state)
    // No walking!
  );
};

const dog = createDog('Rex');
dog.walk();  // Works
dog.swim();  // Works

const fish = createFish('Nemo');
fish.swim();  // Works
// fish.walk();  // Doesn't exist - type safe!
```

### Factory Functions vs Classes

```javascript
// Factory function (more flexible)
function createPerson(name, age) {
  // Private variables (true privacy via closure)
  let _age = age;
  
  return {
    name,
    
    // Public method
    greet() {
      return `Hi, I'm ${this.name}`;
    },
    
    // Getter
    getAge() {
      return _age;
    },
    
    // Setter with validation
    setAge(newAge) {
      if (newAge < 0) throw new Error('Age cannot be negative');
      _age = newAge;
    },
    
    // Method that uses private variable
    canDrink() {
      return _age >= 21;
    }
  };
}

const person = createPerson('Alice', 25);
person.getAge();  // 25
person._age;  // undefined - truly private!

// Class (more familiar to OOP developers)
class Person {
  #age;  // Private field (ES2022)
  
  constructor(name, age) {
    this.name = name;
    this.#age = age;
  }
  
  greet() {
    return `Hi, I'm ${this.name}`;
  }
  
  getAge() {
    return this.#age;
  }
  
  setAge(newAge) {
    if (newAge < 0) throw new Error('Age cannot be negative');
    this.#age = newAge;
  }
  
  canDrink() {
    return this.#age >= 21;
  }
}

// When to use which?
// Factory: Need true privacy, composition, functional style
// Class: Need inheritance, instanceof checks, familiar OOP patterns
```

---

## TypeScript Enhancements

### Proper Type Safety

```typescript
// ❌ Bad: Using 'any'
function processData(data: any) {
  return data.value;  // No type safety
}

// ✅ Good: Proper types
interface Data {
  value: string;
  count: number;
}

function processData(data: Data): string {
  return data.value;  // Type safe
}

// ✅ Better: Generics for reusability
function processData<T extends { value: string }>(data: T): string {
  return data.value;
}
```

### Access Modifiers

```typescript
class BankAccount {
  // Public (default)
  public accountNumber: string;
  
  // Protected (accessible in subclasses)
  protected balance: number;
  
  // Private (only in this class)
  private pin: string;
  
  // Readonly (cannot be changed after initialization)
  readonly createdAt: Date;
  
  constructor(accountNumber: string, initialBalance: number, pin: string) {
    this.accountNumber = accountNumber;
    this.balance = initialBalance;
    this.pin = pin;
    this.createdAt = new Date();
  }
  
  // Public method
  public deposit(amount: number): void {
    this.balance += amount;
  }
  
  // Private method
  private validatePin(inputPin: string): boolean {
    return this.pin === inputPin;
  }
  
  // Public method using private method
  public withdraw(amount: number, inputPin: string): boolean {
    if (!this.validatePin(inputPin)) {
      return false;
    }
    
    if (this.balance >= amount) {
      this.balance -= amount;
      return true;
    }
    
    return false;
  }
  
  // Protected method (for subclasses)
  protected getBalance(): number {
    return this.balance;
  }
}

class SavingsAccount extends BankAccount {
  private interestRate: number;
  
  constructor(accountNumber: string, initialBalance: number, pin: string, interestRate: number) {
    super(accountNumber, initialBalance, pin);
    this.interestRate = interestRate;
  }
  
  public addInterest(): void {
    // Can access protected member
    const interest = this.getBalance() * this.interestRate;
    this.deposit(interest);
    
    // Cannot access private members
    // this.pin;  // Error!
  }
}
```

### Interfaces vs Abstract Classes

```typescript
// Interface: Contract for structure (no implementation)
interface ILogger {
  log(message: string): void;
  error(message: string): void;
}

// Can implement multiple interfaces
class ConsoleLogger implements ILogger {
  log(message: string): void {
    console.log(message);
  }
  
  error(message: string): void {
    console.error(message);
  }
}

// Abstract class: Partial implementation
abstract class BaseLogger {
  // Concrete method (shared implementation)
  protected formatMessage(message: string): string {
    return `[${new Date().toISOString()}] ${message}`;
  }
  
  // Abstract method (must be implemented by subclass)
  abstract log(message: string): void;
  abstract error(message: string): void;
}

class FileLogger extends BaseLogger {
  private filePath: string;
  
  constructor(filePath: string) {
    super();
    this.filePath = filePath;
  }
  
  log(message: string): void {
    const formatted = this.formatMessage(message);
    // Write to file...
  }
  
  error(message: string): void {
    const formatted = this.formatMessage(message);
    // Write error to file...
  }
}

// When to use which?
// Interface: Multiple inheritance, pure contract, no shared code
// Abstract class: Single inheritance, shared implementation, template method pattern
```

### Utility Types

```typescript
// Partial: Make all properties optional
interface User {
  id: number;
  name: string;
  email: string;
}

function updateUser(id: number, updates: Partial<User>) {
  // updates can have any subset of User properties
}

updateUser(1, { name: 'Alice' });  // Valid
updateUser(1, { email: 'alice@example.com' });  // Valid

// Required: Make all properties required
type RequiredUser = Required<Partial<User>>;

// Readonly: Make all properties readonly
type ReadonlyUser = Readonly<User>;

// Pick: Select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit: Exclude specific properties
type UserWithoutId = Omit<User, 'id'>;

// Record: Create object type with specific keys
type UserRoles = Record<'admin' | 'user' | 'guest', boolean>;

// ReturnType: Get return type of function
function getUser(): User {
  return { id: 1, name: 'Alice', email: 'alice@example.com' };
}

type UserType = ReturnType<typeof getUser>;  // User
```

---

## Common Pitfalls

### 1. Losing `this` Context

```javascript
// ❌ Problem
class Counter {
  count = 0;
  
  increment() {
    this.count++;
  }
}

const counter = new Counter();
const incrementFn = counter.increment;
incrementFn();  // Error! 'this' is undefined

// ✅ Solution 1: Arrow function
class Counter {
  count = 0;
  
  increment = () => {
    this.count++;
  }
}

// ✅ Solution 2: Bind in constructor
class Counter {
  count = 0;
  
  constructor() {
    this.increment = this.increment.bind(this);
  }
  
  increment() {
    this.count++;
  }
}

// ✅ Solution 3: Arrow function when passing
const counter = new Counter();
setTimeout(() => counter.increment(), 1000);
```

### 2. Mutating Objects

```javascript
// ❌ Bad: Mutation
const user = { name: 'Alice', age: 25 };
function updateAge(user, newAge) {
  user.age = newAge;  // Mutates original!
  return user;
}

// ✅ Good: Immutability
function updateAge(user, newAge) {
  return { ...user, age: newAge };  // New object
}

// ✅ Better: Immutability with TypeScript
interface User {
  readonly name: string;
  readonly age: number;
}

function updateAge(user: Readonly<User>, newAge: number): User {
  return { ...user, age: newAge };
}
```

### 3. Not Using `const` and `let`

```javascript
// ❌ Bad: Using var
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // 3, 3, 3
}

// ✅ Good: Using let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);  // 0, 1, 2
}

// ✅ Best: Use const by default
const users = ['Alice', 'Bob'];
users.push('Charlie');  // OK - array is mutable
// users = [];  // Error - cannot reassign const
```

### 4. Not Handling Async Properly

```javascript
// ❌ Bad: Callback hell
getData(function(a) {
  getMoreData(a, function(b) {
    getMoreData(b, function(c) {
      getMoreData(c, function(d) {
        // ...
      });
    });
  });
});

// ✅ Good: Promises
getData()
  .then(a => getMoreData(a))
  .then(b => getMoreData(b))
  .then(c => getMoreData(c))
  .catch(err => console.error(err));

// ✅ Better: Async/await
async function fetchData() {
  try {
    const a = await getData();
    const b = await getMoreData(a);
    const c = await getMoreData(b);
    return c;
  } catch (err) {
    console.error(err);
  }
}

// ✅ Best: Parallel execution when possible
async function fetchAllData() {
  const [users, posts, comments] = await Promise.all([
    fetchUsers(),
    fetchPosts(),
    fetchComments()
  ]);
  
  return { users, posts, comments };
}
```

### 5. Memory Leaks

```javascript
// ❌ Bad: Event listener not removed
class Component {
  constructor() {
    window.addEventListener('resize', this.handleResize);
  }
  
  handleResize = () => {
    // Handle resize
  }
  
  // Component destroyed but listener still exists!
}

// ✅ Good: Clean up
class Component {
  constructor() {
    window.addEventListener('resize', this.handleResize);
  }
  
  handleResize = () => {
    // Handle resize
  }
  
  destroy() {
    window.removeEventListener('resize', this.handleResize);
  }
}

// ✅ Better: AbortController (modern)
class Component {
  controller = new AbortController();
  
  constructor() {
    window.addEventListener('resize', this.handleResize, {
      signal: this.controller.signal
    });
  }
  
  handleResize = () => {
    // Handle resize
  }
  
  destroy() {
    this.controller.abort();  // Removes all listeners
  }
}
```

---

## Best Practices

### 1. SOLID Principles in JavaScript

```typescript
// S - Single Responsibility Principle
// ❌ Bad: Class does too much
class User {
  constructor(public name: string, public email: string) {}
  
  save() {
    // Database logic
  }
  
  sendEmail() {
    // Email logic
  }
  
  generateReport() {
    // Report logic
  }
}

// ✅ Good: Separate responsibilities
class User {
  constructor(public name: string, public email: string) {}
}

class UserRepository {
  save(user: User) {
    // Database logic
  }
}

class EmailService {
  send(user: User, message: string) {
    // Email logic
  }
}

class ReportGenerator {
  generate(user: User) {
    // Report logic
  }
}

// O - Open/Closed Principle
// ❌ Bad: Modifying existing code for new features
class PaymentProcessor {
  process(type: string, amount: number) {
    if (type === 'credit') {
      // Process credit card
    } else if (type === 'paypal') {
      // Process PayPal
    } else if (type === 'crypto') {  // Adding new type requires modification
      // Process crypto
    }
  }
}

// ✅ Good: Open for extension, closed for modification
interface PaymentMethod {
  process(amount: number): void;
}

class CreditCardPayment implements PaymentMethod {
  process(amount: number): void {
    // Process credit card
  }
}

class PayPalPayment implements PaymentMethod {
  process(amount: number): void {
    // Process PayPal
  }
}

class CryptoPayment implements PaymentMethod {
  process(amount: number): void {
    // Process crypto
  }
}

class PaymentProcessor {
  constructor(private method: PaymentMethod) {}
  
  process(amount: number): void {
    this.method.process(amount);
  }
}

// L - Liskov Substitution Principle
// ❌ Bad: Subclass changes behavior unexpectedly
class Rectangle {
  constructor(protected width: number, protected height: number) {}
  
  setWidth(width: number) {
    this.width = width;
  }
  
  setHeight(height: number) {
    this.height = height;
  }
  
  getArea(): number {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  setWidth(width: number) {
    this.width = width;
    this.height = width;  // Breaks LSP!
  }
  
  setHeight(height: number) {
    this.width = height;
    this.height = height;  // Breaks LSP!
  }
}

// ✅ Good: Proper abstraction
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  
  getArea(): number {
    return this.width * this.height;
  }
}

class Square implements Shape {
  constructor(private side: number) {}
  
  getArea(): number {
    return this.side * this.side;
  }
}

// I - Interface Segregation Principle
// ❌ Bad: Fat interface
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Robot implements Worker {
  work() { /* ... */ }
  eat() { /* Robots don't eat! */ }
  sleep() { /* Robots don't sleep! */ }
}

// ✅ Good: Segregated interfaces
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
  work() { /* ... */ }
  eat() { /* ... */ }
  sleep() { /* ... */ }
}

class Robot implements Workable {
  work() { /* ... */ }
}

// D - Dependency Inversion Principle
// ❌ Bad: High-level depends on low-level
class MySQLDatabase {
  save(data: any) {
    // MySQL specific code
  }
}

class UserService {
  private db = new MySQLDatabase();  // Tightly coupled!
  
  saveUser(user: User) {
    this.db.save(user);
  }
}

// ✅ Good: Depend on abstractions
interface Database {
  save(data: any): void;
}

class MySQLDatabase implements Database {
  save(data: any) {
    // MySQL specific code
  }
}

class PostgreSQLDatabase implements Database {
  save(data: any) {
    // PostgreSQL specific code
  }
}

class UserService {
  constructor(private db: Database) {}  // Dependency injection
  
  saveUser(user: User) {
    this.db.save(user);
  }
}

// Usage
const mysqlDb = new MySQLDatabase();
const userService = new UserService(mysqlDb);

// Easy to swap
const postgresDb = new PostgreSQLDatabase();
const userService2 = new UserService(postgresDb);
```

### 2. Prefer Functional Patterns

```typescript
// ❌ Imperative (how to do it)
const numbers = [1, 2, 3, 4, 5];
const doubled = [];
for (let i = 0; i < numbers.length; i++) {
  doubled.push(numbers[i] * 2);
}

// ✅ Declarative (what to do)
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(n => n * 2);

// ✅ Functional composition
const add = (a: number) => (b: number) => a + b;
const multiply = (a: number) => (b: number) => a * b;

const add5 = add(5);
const double = multiply(2);

const result = [1, 2, 3]
  .map(add5)      // [6, 7, 8]
  .map(double);   // [12, 14, 16]
```

### 3. Use TypeScript Strictly

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true
  }
}

// ❌ Bad
function getUser(id) {  // Implicit any
  return users.find(u => u.id === id);  // May return undefined
}

// ✅ Good
function getUser(id: number): User | undefined {
  return users.find(u => u.id === id);
}

// ✅ Better: Handle null/undefined explicitly
function getUser(id: number): User {
  const user = users.find(u => u.id === id);
  if (!user) {
    throw new Error(`User ${id} not found`);
  }
  return user;
}
```

---

## Design Patterns

### Singleton

```typescript
// ❌ Bad: Global variable
let instance: Database;

export function getDatabase() {
  if (!instance) {
    instance = new Database();
  }
  return instance;
}

// ✅ Good: Proper singleton
class Database {
  private static instance: Database;
  
  private constructor() {
    // Private constructor prevents direct instantiation
  }
  
  public static getInstance(): Database {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }
  
  public query(sql: string) {
    // ...
  }
}

// Usage
const db = Database.getInstance();
```

### Factory

```typescript
interface Animal {
  speak(): string;
}

class Dog implements Animal {
  speak() {
    return 'Woof!';
  }
}

class Cat implements Animal {
  speak() {
    return 'Meow!';
  }
}

// Factory
class AnimalFactory {
  static create(type: 'dog' | 'cat'): Animal {
    switch (type) {
      case 'dog':
        return new Dog();
      case 'cat':
        return new Cat();
      default:
        throw new Error(`Unknown animal type: ${type}`);
    }
  }
}

// Usage
const dog = AnimalFactory.create('dog');
dog.speak();  // 'Woof!'
```

### Observer (Pub/Sub)

```typescript
type Listener<T> = (data: T) => void;

class EventEmitter<T = any> {
  private listeners: Map<string, Set<Listener<T>>> = new Map();
  
  on(event: string, listener: Listener<T>): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    
    this.listeners.get(event)!.add(listener);
    
    // Return unsubscribe function
    return () => this.off(event, listener);
  }
  
  off(event: string, listener: Listener<T>): void {
    this.listeners.get(event)?.delete(listener);
  }
  
  emit(event: string, data: T): void {
    this.listeners.get(event)?.forEach(listener => listener(data));
  }
}

// Usage
const emitter = new EventEmitter<{ userId: number }>();

const unsubscribe = emitter.on('user:login', (data) => {
  console.log(`User ${data.userId} logged in`);
});

emitter.emit('user:login', { userId: 123 });

unsubscribe();  // Clean up
```

### Dependency Injection

```typescript
// Define dependencies
interface Logger {
  log(message: string): void;
}

interface Database {
  query(sql: string): Promise<any>;
}

// Implementations
class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
}

class PostgresDatabase implements Database {
  async query(sql: string): Promise<any> {
    // Execute query
  }
}

// Service with dependencies
class UserService {
  constructor(
    private logger: Logger,
    private database: Database
  ) {}
  
  async getUser(id: number) {
    this.logger.log(`Fetching user ${id}`);
    return this.database.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// Manual DI
const logger = new ConsoleLogger();
const database = new PostgresDatabase();
const userService = new UserService(logger, database);

// Or use DI container (e.g., InversifyJS, TypeDI)
```

---

## Summary

### Key Takeaways

1. **Understand prototypes** - Classes are syntactic sugar
2. **Master `this` binding** - Know the 5 rules
3. **Use closures wisely** - For privacy and state management
4. **Prefer composition** over inheritance
5. **Follow SOLID principles** - Write maintainable code
6. **Use TypeScript strictly** - Catch errors at compile time
7. **Embrace functional patterns** - Immutability, pure functions
8. **Clean up resources** - Prevent memory leaks
9. **Handle async properly** - async/await over callbacks
10. **Apply design patterns** - When they solve real problems

### Recommended Reading

- [You Don't Know JS](https://github.com/getify/You-Dont-Know-JS)
- [JavaScript: The Good Parts](https://www.oreilly.com/library/view/javascript-the-good/9780596517748/)
- [Effective TypeScript](https://effectivetypescript.com/)
- [Clean Code JavaScript](https://github.com/ryanmcdermott/clean-code-javascript)

---

**Last Updated**: 2026-01-28  
**Next Review**: 2026-04-28
