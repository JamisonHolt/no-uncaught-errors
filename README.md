# eslint-plugin-no-uncaught-errors

A comprehensive ESLint plugin that enforces proper error documentation and handling through JSDoc `@throws` annotations. This plugin helps create safer, more predictable codebases by requiring explicit documentation of all possible errors and flagging unhandled error cases.

## Philosophy

This plugin is designed as a lightweight alternative to comprehensive error handling libraries like `neverthrow` or `Effect`. Instead of requiring structural changes to your code, it leverages mathematical induction: if every function declares its possible errors via `@throws` JSDoc tags, then the entire codebase has predictable error behavior.

The plugin operates on the principle that **explicit is better than implicit** when it comes to error handling, even if it means some additional boilerplate.

## Installation

```bash
npm install --save-dev eslint-plugin-no-uncaught-errors
```

## Configuration

Add the plugin to your ESLint configuration:

```javascript
// .eslintrc.js
module.exports = {
  plugins: ['no-uncaught-errors'],
  rules: {
    'no-uncaught-errors/require-throws': 'error'
  }
}
```

## Rule: `require-throws`

This rule enforces that all functions declare their possible errors via `@throws` JSDoc annotations.

### Basic Usage

```javascript
// ✅ Good - explicitly declares what it throws
/**
 * Validates user input
 * @throws {ValidationError} When input is invalid
 * @throws {TypeError} When input is not a string
 */
function validateInput(input) {
  if (typeof input !== 'string') {
    throw new TypeError('Input must be a string');
  }
  if (!input.trim()) {
    throw new ValidationError('Input cannot be empty');
  }
}

// ✅ Good - explicitly declares it throws nothing
/**
 * Adds two numbers
 * @throws {never}
 */
function add(a, b) {
  return a + b;
}

// ❌ Bad - no @throws declaration
function processData(data) {
  return JSON.parse(data); // JSON.parse can throw!
}
```

### Configuration Options

```javascript
{
  "no-uncaught-errors/require-throws": ["error", {
    "requireNever": true,         // Require @throws {never} for non-throwing functions
    "allowErrorBubbling": true,   // Allow uncaught errors if declared in JSDoc
    "unsafeCalls": "warn",        // How to handle calls to undocumented functions
    "errorHandlers": ["tryCatch", "withErrorHandling"], // Functions that handle errors
    "genericWrappers": ["withRetries", "withTimeout"], // Functions that pass through errors
    "strictMode": false           // Require specific error types (no generic Error)
  }]
}
```

#### `requireNever` (default: `true`)

When `true`, functions that don't throw must explicitly declare `@throws {never}`.

```javascript
// With requireNever: true
/**
 * @throws {never}
 */
function safeMath(a, b) {
  return a + b; // No throws needed
}

// With requireNever: false
function safeMath(a, b) {
  return a + b; // @throws {never} not required
}
```

#### `allowErrorBubbling` (default: `true`)

When `true`, functions can let errors bubble up if they declare them in their `@throws`.

```javascript
// ✅ Allowed with allowErrorBubbling: true
/**
 * @throws {SyntaxError} From JSON.parse when invalid JSON
 */
function parseConfig(jsonString) {
  return JSON.parse(jsonString); // Error bubbles up but is declared
}

// ❌ Required with allowErrorBubbling: false
/**
 * @throws {ConfigError} When parsing fails
 */
function parseConfig(jsonString) {
  try {
    return JSON.parse(jsonString);
  } catch (error) {
    throw new ConfigError('Failed to parse config', { cause: error });
  }
}
```

#### `unsafeCalls` (default: `"warn"`)

Controls how to handle calls to functions without `@throws` documentation (typically third-party libraries).

- `"error"`: Treat as errors
- `"warn"`: Treat as warnings  
- `"off"`: Ignore unsafe calls

```javascript
// With unsafeCalls: "warn"
function processData(data) {
  // ⚠️ Warning: JSON.parse has no @throws documentation
  return JSON.parse(data);
}

// Recommended approach:
/**
 * @throws {SyntaxError} When JSON is invalid
 */
function parseJSON(data) {
  return JSON.parse(data);
}
```

#### `errorHandlers` (default: `[]`)

List of function names that are recognized as error handlers. Calls within these functions are considered "handled".

```javascript
// Configuration
{
  "errorHandlers": ["tryCatch", "withErrorHandling"]
}

// Usage
function riskyOperation() {
  // ✅ Considered handled because it's inside tryCatch
  const [result, error] = tryCatch(() => JSON.parse(data));
  if (error) {
    // handle error
  }
  return result;
}
```

#### `genericWrappers` (default: `[]`)

List of function names that wrap other functions and pass through their errors unchanged. These functions inherit the `@throws` signature of their wrapped function.

```javascript
// Configuration
{
  "genericWrappers": ["withRetries", "withTimeout", "withLogging"]
}

// Usage - withRetries passes through the original function's errors
/**
 * Retries a function up to 3 times with exponential backoff
 * @template T - The wrapped function type
 * @throws {T} All errors from the wrapped function (after retries exhausted)
 */
async function withRetries(fn, maxRetries = 3) {
  let lastError;
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (i < maxRetries - 1) {
        await sleep(Math.pow(2, i) * 1000 + Math.random() * 1000);
      }
    }
  }
  throw lastError;
}

// The linter understands that this throws ValidationError | NetworkError
/**
 * @throws {ValidationError} From input validation in apiCall
 * @throws {NetworkError} From network request in apiCall
 */
async function reliableApiCall(data) {
  return await withRetries(() => apiCall(data)); // apiCall throws ValidationError | NetworkError
}
```

#### `strictMode` (default: `false`)

When `true`, requires specific error types instead of generic `Error` class.

```javascript
// With strictMode: true
/**
 * ✅ Good - specific error types
 * @throws {ValidationError} When validation fails
 * @throws {NetworkError} When network request fails
 */
function processUser(data) {
  // ...
}

/**
 * ❌ Bad - generic Error type
 * @throws {Error} When something goes wrong
 */
function processUser(data) {
  // ...
}
```

## Arrow Functions and JSDoc

Arrow functions can be documented using JSDoc comments placed before the assignment:

```javascript
// ✅ Good - arrow function with JSDoc
/**
 * @throws {ValidationError} When input is invalid
 */
const validateUser = (user) => {
  if (!user.email) throw new ValidationError('Email required');
  return user;
};

// ❌ Bad - arrow function without @throws but calls throwing function
const processUser = (user) => {
  return validateUser(user); // Must handle or declare ValidationError
};
```

If an arrow function cannot be documented with JSDoc (e.g., inline callbacks), it must handle all errors internally or the parent function must declare them:

```javascript
/**
 * @throws {ValidationError} From user validation in callback
 */
function processUsers(users) {
  // Arrow function errors must be declared by parent
  return users.map(user => validateUser(user));
}
```

## Class Methods and Constructors

Class methods and constructors follow the same rules as regular functions:

```javascript
class UserService {
  /**
   * @throws {DatabaseError} When connection fails
   */
  constructor(dbConfig) {
    this.db = new Database(dbConfig); // May throw DatabaseError
  }

  /**
   * @throws {ValidationError} When user data is invalid
   * @throws {DatabaseError} When database save fails
   */
  async createUser(userData) {
    this.validateUser(userData); // throws ValidationError
    return await this.db.save(userData); // throws DatabaseError
  }

  /**
   * @throws {never}
   */
  formatUserName(user) {
    return `${user.first} ${user.last}`;
  }
}

// Usage must handle constructor errors
/**
 * @throws {DatabaseError} When UserService construction fails
 */
function initializeService() {
  return new UserService(config); // Constructor can throw
}
```

## Multiple @throws vs Union Types

Both syntaxes are supported, but multiple `@throws` tags are preferred as they allow inline documentation of when each error occurs:

```javascript
// ✅ Preferred - multiple @throws tags with context
/**
 * @throws {ValidationError} When validation fails
 * @throws {NetworkError} When network request fails  
 * @throws {TimeoutError} When operation times out
 */
function complexOperation() {
  // ...
}

// ✅ Also valid - union types (auto-corrected to multiple tags)
/**
 * @throws {ValidationError | NetworkError | TimeoutError}
 */
function complexOperation() {
  // ...
}
```

## Generic Wrapper Functions

For utility functions that wrap other functions (common in AI/retry scenarios), use the `genericWrappers` configuration:

```javascript
/**
 * Wraps function with timeout capability
 * @template T - The wrapped function's error type
 * @throws {TimeoutError} When operation exceeds timeout
 * @throws {T} All errors from wrapped function
 */
async function withTimeout(fn, ms) {
  return Promise.race([
    fn(),
    new Promise((_, reject) => 
      setTimeout(() => reject(new TimeoutError()), ms)
    )
  ]);
}

/**
 * Adds logging to function calls
 * @template T - The wrapped function's error type  
 * @throws {T} All errors from wrapped function (after logging)
 */
async function withLogging(fn, context) {
  try {
    console.log(`Starting ${context}`);
    const result = await fn();
    console.log(`Completed ${context}`);
    return result;
  } catch (error) {
    console.error(`Failed ${context}:`, error);
    throw error; // Re-throw original error
  }
}

// Complex composition example
/**
 * @throws {ValidationError} From input validation in OpenAI API
 * @throws {NetworkError} From network request to OpenAI
 * @throws {TimeoutError} When API call exceeds 30 seconds
 */
async function robustAICall(prompt) {
  return await withTimeout(
    () => withRetries(
      () => withLogging(
        () => openai.complete(prompt), // throws ValidationError | NetworkError
        'AI completion'
      )
    ),
    30000
  );
}
```

## Auto-fixing

The plugin provides auto-fix capabilities for common scenarios:

- Adds missing `@throws {never}` to functions that don't throw
- Propagates `@throws` declarations up the call chain when `allowErrorBubbling` is enabled
- Converts union type syntax to multiple `@throws` tags
- Infers error types for generic wrapper functions
- Suggests wrapping unsafe third-party calls

## Advanced Examples

### Error Propagation with Classes

```javascript
class PaymentProcessor {
  /**
   * @throws {ValidationError} When payment data is invalid
   * @throws {PaymentError} When card processing fails
   * @throws {NetworkError} When payment gateway is unreachable
   */
  async processPayment(paymentData) {
    this.validatePayment(paymentData); // throws ValidationError
    await this.chargeCard(paymentData); // throws PaymentError | NetworkError
  }
}

/**
 * @throws {ValidationError} From payment validation
 * @throws {PaymentError} From payment processing 
 * @throws {NetworkError} From payment gateway communication
 */
async function handleCheckout(orderData) {
  const processor = new PaymentProcessor();
  return await processor.processPayment(orderData.payment);
}
```

### AI/Retry Patterns

```javascript
/**
 * @throws {ValidationError} When prompt validation fails
 * @throws {NetworkError} When API request fails
 * @throws {RateLimitError} When rate limit is exceeded
 * @throws {TimeoutError} When request times out
 */
async function smartAICall(prompt, options = {}) {
  // Composition of multiple wrappers
  return await withTimeout(
    () => withRetries(
      () => withRateLimit(
        () => openai.complete(prompt, options),
        'openai-api'
      ),
      { maxRetries: 3, backoff: 'exponential' }
    ),
    options.timeout || 30000
  );
}
```

## Benefits

1. **Mathematical Induction**: If every function declares its errors, the entire codebase has predictable error behavior
2. **Zero Runtime Overhead**: Pure static analysis with no impact on production code
3. **Language Agnostic**: Works with plain JavaScript - no TypeScript required
4. **Gradual Adoption**: Can be enabled incrementally with warning levels
5. **Rich Documentation**: Multiple `@throws` tags provide contextual error information
6. **Tooling Integration**: Works with existing JSDoc tooling and IDE support
7. **Class Support**: Full support for class methods, constructors, and inheritance
8. **Flexible Syntax**: Supports both union types and multiple @throws tags
9. **Generic Function Support**: Handles wrapper functions that pass through errors

## Comparison to Alternatives

| Feature | eslint-plugin-no-uncaught-errors | neverthrow | Effect |
|---------|----------------------------------|------------|--------|
| Runtime overhead | None | Some | Some |
| Code structure changes | None | Moderate | Extensive |
| Enforcement | Lint-time | None | Type-system |
| Learning curve | Low | Medium | High |
| Existing code integration | Easy | Moderate | Difficult |
| Language support | JavaScript/TypeScript | JavaScript/TypeScript | TypeScript only |
| Class/OOP support | Full | Limited | Limited |
| Arrow function support | Full | Full | Limited |
| Generic wrapper support | Full | Partial | Full |
| Error composition patterns | Basic | Good | Excellent |
| Functional programming benefits | None | Some | Full |

Both `neverthrow` and `Effect` are excellent libraries that offer different approaches to error handling. Choose the tool that best fits your team's preferences, existing codebase, and architectural goals.

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT