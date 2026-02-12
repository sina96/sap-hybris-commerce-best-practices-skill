# Code Conventions

This skill enforces the coding standards and style guidelines for SAP Hybris development, derived from the official IntelliJ IDEA code style configuration.

## Indentation & Formatting
- **Indentation**: Use **Tabs** (not spaces).
- **Tab Size**: 3
- **Indent Size**: 3
- **Right Margin**: 130 characters

## Control Flow & Braces
- **Else on new line**: `true` (e.g., `} \n else {`)
- **While on new line**: `true` (e.g., `} \n while (...)`)
- **Catch on new line**: `true`
- **Finally on new line**: `true`
- **Brace Style**: End of line (standard Java) but with keywords on new lines as above.

### Example
```java
try {
   doSomething();
}
catch (Exception e) {
   handleError();
}
finally {
   cleanup();
}

if (condition) {
   // ...
}
else {
   // ...
}
```

## Imports
- **Class count to use import on demand (`*`)**: 20 (Avoid `*` imports unless > 20)
- **Import Layout Order**:
  1. `java.*`
  2. `javax.*`
  3. `org.*`
  4. `com.*`
  5. All other imports
- Separate these groups with blank lines.

## Wrapping
- **Binary operations**: Wrap if long, place operator on **next line**.
- **Method parameters**: Wrap if long.
