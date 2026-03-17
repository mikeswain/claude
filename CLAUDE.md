# Tool Usage
- Never use `cd <path> && git ...` compound commands. Use `git -C <path>` instead to avoid compound command approval prompts.
- Prefer single-command forms over `cd && <tool>` patterns generally.

# Coding Preferences (All Languages)
- Readability and maintainability over cleverness
- The best code is no code; avoid unnecessary complexity and boilerplate
- Descriptive names chosen carefully to convey intent and domain meaning
- Follow the language's common case conventions
- Avoid deep nesting; functions over ~20 lines are getting uncomfortable
- Prefer immutable, functional code — easier to debug and prove correct
- Scope state and symbols as narrowly as possible; avoid global state and side effects
- Fail fast with runtime exceptions; prefer assertions over labouring on vanishingly unlikely cases
- Never swallow exceptions unless there is a specific mitigation path (document with a comment)
- Avoid complex nested if/then and break/continue unless it genuinely clarifies the logic
- Prefer static imports; dynamic imports only when genuinely justified by download size

# Testing
- **Write tests proactively alongside new logic — do not wait to be asked**
- Unit tests for all public functions and critical internal logic
- Descriptive test names that convey the scenario
- Meaningful tests over coverage percentage
- Use test doubles to isolate units, but don't mock stable framework/project dependencies
- Prefer integration tests for multi-component end-to-end scenarios

# Documentation
- Clear, concise docstrings for all public APIs (language-specific format in skills)
- Examples in docstrings where they clarify usage
- Inline comments only for non-obvious decisions; prefer self-explanatory code

# Refactoring
- All tests pass before, during, and after
- Small, incremental changes over large sweeping ones
