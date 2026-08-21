Reused rules in CLAUDE.md

## Code Review Standards
After completing any implementation, review the code for:
- Functions longer than 30 lines (likely doing too much)
- Logic duplicated more than twice (extract to utility)
- Missing type hints on any function parameter or return type
- Bare `except:` clauses (must catch specific exceptions)
- Functions with more than 3 parameters that could be grouped into a dict/dataclass
- Run /simplify before presenting code to the user.

## Python Style

- **Line length:** 80 characters
- **Quotes:** Single quotes; double only when the string contains a single quote
- **Type hints:** Always on all function parameters and return types
- **Naming:** `lowerCamelCase` for functions and variables; logical short-form abbreviations (concept + qualifier, e.g. `capMkt`, `epsAnnual`, `priceWeekly`); `PascalCase` for classes; `UPPER_SNAKE` for module-level constants
- **Docstrings:** Multi-line prose when purpose is not obvious; 20-line max; no formal Args/Returns sections
- **Imports:** stdlib first, then local; blank line between groups
- **String formatting:** f-strings preferred over `.format()` or `%`
- **Error handling:** Always catch specific exceptions, never bare `except:`

## Rules

- in plan mode never edit without asking, aim is to just plan the needed steps
- Always ask clarifying questions before starting a complex task
- Show your plan and steps before executing
- cite sources when doing research
- do not guess, ask clarifying questions first
- in code never use Umlaute istead use `ae`, `ue`, `oe`
