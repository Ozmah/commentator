---
name: commentator
description: |
  Write and edit code comments that earn their place: what the code cannot say, contracts the
  signature does not carry, and effects that happen somewhere else. Use when writing or editing
  code, and when reviewing or cleaning up comments: field-by-field docs on types and classes,
  comments that restate a name or the next line, docblocks on trivial functions, types repeated
  in docs, or comments that justify a decision to whoever reviewed the session.
license: MIT
metadata:
  version: "0.1.0"
---

# Commentator: comments that earn their place

A comment is text that no compiler or test checks, and it drifts when the code around it changes. Each one should give the next reader something the code cannot. Write fewer comments, and make the ones you keep carry that.

The examples use TypeScript and PHP. The principles apply to any language.

## Why agent code is over-commented

An agent writes code inside a conversation, and its comments drift toward that conversation. They justify a choice to the person reviewing the session ("so a stray click never approves"), restate what was just decided, or label every field because labeling feels thorough. The next reader was not in that conversation. They need the code and the few things the code cannot tell them.

## The test

Before writing or keeping a comment, ask: would a competent reader, with only the code in front of them, get this wrong or lose time without it?

If the answer is no, leave the comment out. If it is yes, write it short and accurate.

This skill is about comments. It never changes the code they describe: no renames, extracted functions, new constants or restructuring, even when a comment seems to ask for one. The code is the user's decision, and it may belong to someone else, be a public API or follow conventions you cannot see from one file.

A competent reader knows the project's language and framework. Do not explain how the framework works: a Laravel reader knows that `Cache::remember()` goes through a facade and that `$user->posts` loads a relation. Do explain what the framework does here that the line does not show.

## What a comment is for

Write a comment, and keep one, when it says one of these:

1. Why, when the code cannot show it: a constraint from a library, a framework, a protocol, a shell or an operating system.
2. What must not change, and what breaks if it does: ordering, invariants, security boundaries.
3. Where something else lives. The policy is in another module, and this code only maps it.
4. What happens somewhere else when this runs: an observer that sends mail, a global scope that filters rows, an event listener, a lazy-loaded relation. The reader of this line cannot see them, and they change what it does.
5. A contract the signature does not carry: what a function accepts beyond its types, what it returns in edge cases, what it throws.
6. What a tool requires: a `SAFETY:` justification a linter demands, a suppression with its reason, a `TODO` with enough context to act on.

A good example, from a request middleware:

```ts
/**
 * Maintenance gate driven by MAINTENANCE_MODE (Railway service variable).
 * Policy lives in `@/lib/maintenance`; this only maps decisions to responses.
 * Runs after the observability boundary so blocked requests are still logged.
 */
const maintenanceMiddleware = createMiddleware().server(async ({ pathname, request, handlerType, next }) => {
	// ...
	case "render": {
		const result = await next();
		// ...
		// Without the ownership transfer Start treats this as a replaced
		// response and aborts the SSR stream it is still writing.
		return transferResponseBodyOwnership(result.response, new Response(result.response.body, { status: 503, headers }));
	}
});
```

The docblock gives the configuration source (1), where the policy lives (3) and why the order matters (2). The inline comment explains framework behavior (1) and protects a call that looks removable (2).

Effects that happen somewhere else (4) belong at the call site where they matter:

```php
// The UserObserver sends the welcome email on create; seeders use createQuietly().
$user = User::create($data);

// Includes soft-deleted orders: the report must match the invoice totals.
$orders = Order::withTrashed()->whereBelongsTo($customer)->get();
```

The first prevents a duplicate email, the second a report whose totals do not match.

## How to work

### Writing new code

Add only the comments the test keeps. Most fields, most private methods and most short functions need none.

### Editing existing code

Apply this to the comments in the code you touch, not to whole files. Remove a comment only when it fails the test. When in doubt, and always near security or concurrency, keep it. Never remove a comment a linter or another tool requires.

### Cleaning up on request

Go one file at a time. For each comment, decide whether to keep it, rewrite it or remove it, and change comments only. After each file, give the report below and wait for the user's go-ahead before the next one.

When writing or editing code for any other task, give no report: the comments are part of the change.

### The report, per file

1. **The diff** of the file, comments only.
2. **The counts:** comments removed, rewritten, kept on merit, and kept because a tool requires them (`SAFETY:`, suppressions, annotations). Then the comment lines before and after: "comment lines: 48 → 21".
3. **Each removed or rewritten comment**, with the pattern that justified it ("pattern 3, field note") and a confidence level. List the medium and low ones first, so the user knows where to look.
4. **Special cases left untouched:** each comment you could not judge, quoted, with what is missing to decide. Usually it describes a constraint outside the file or the project that you cannot check.
5. **Verification** that only comments changed. Compare the file before and after with its comments stripped: the two must be identical. Then run what the project has, such as the formatter, the type checker or the tests. Say what ran and its result, or that nothing could be checked and why.

Confidence levels:

- **High:** the pattern clearly applies, and nothing outside the file could make the comment necessary.
- **Medium:** the pattern applies, but the comment might point to something in another file. Say what.
- **Low:** the decision depends on knowledge you do not have. Prefer leaving the comment and listing it as a special case; a low-confidence removal should be rare.

### When the project has a convention

Some projects require a docblock on every public method or every property, or generate docs from them. Follow the convention, and fill it with what the test keeps: one useful line instead of a restated name. Do not fight the convention in a cleanup.

## Patterns to remove

Strongest first. Patterns 1 to 3 justify an edit on one sighting.

### 1. Comments addressed to the session

**Watch for:** justifications of a choice ("so a stray click never approves", "on purpose", "intentionally"), answers to an objection nobody in the code raises, explanations of why an alternative was rejected.

**Problem:** The comment argues with whoever reviewed the session. The next reader needs the constraint and has no use for the debate. Keep the constraint when it is real and drop the argument.

```ts
// Before
/** The option a first click selected; a second click on it confirms, so a stray click never approves (Yes is selected by default). */
private armed: number | undefined;

// After
/** A click selects; a second click on the same option confirms. */
private armed: number | undefined;
```

### 2. Restating the name, the type or the next line

**Watch for:** a field comment that repeats its name in a sentence, a comment that describes the line under it, a `@return` that repeats the function name.

**Problem:** It adds a line and no information. Remove it.

```ts
// Before
/** The option that opened the input. */
private inputKey = TELL.key;
/** Dialog title in non-TUI modes. */
title: string;
// Increment the counter
count++;

// After
private inputKey = TELL.key;
title: string;
count++;
```

```php
// Before
/**
 * Get the user's email.
 *
 * @return string The email.
 */
public function getEmail(): string

// After
public function getEmail(): string
```

### 3. Field-by-field documentation

**Watch for:** a type, interface, class or DTO where most fields carry a comment.

**Problem:** Comments between the fields break up the structure, and labeling every field hides the few notes that matter. Leave the fields bare. When the type needs explaining, put one comment above it: what the type is for, then a list of only the fields whose meaning, unit, default or constraint is not in the name and type.

```ts
// Before
export interface BodyLine {
	/** Plain text. Also used as-is by non-TUI fallbacks when `spans` is set. */
	text: string;
	/** Colored runs that replace `text` in the TUI. */
	spans?: Span[];
	/** Defaults to "text". */
	color?: ThemeColor;
	/** Printed before the first wrapped row, e.g. "$ ", "+ ", "12 │ ". */
	prefix?: string;
	/** Printed before continuation rows. Defaults to spaces as wide as `prefix`. */
	contPrefix?: string;
	prefixColor?: ThemeColor;
}

// After
/**
 * One line of the prompt body, wrapped to the terminal width.
 *
 * - `text`: non-TUI clients print it even when `spans` is set.
 * - `prefix`: on the first wrapped row only, e.g. "$ ", "+ ", "12 │ ".
 * - `contPrefix`: on the other rows; defaults to spaces as wide as `prefix`.
 */
export interface BodyLine {
	text: string;
	spans?: Span[];
	color?: ThemeColor;
	prefix?: string;
	contPrefix?: string;
	prefixColor?: ThemeColor;
}
```

The list keeps only what the names cannot say: who reads `text`, where `prefix` applies, the default of `contPrefix`. `spans`, `color` and `prefixColor` need nothing.

### 4. Docblocks on functions that need none

**Watch for:** a docblock on a short function whose name, parameters and types already say everything: getters, setters, simple predicates, thin wrappers, private helpers.

**Problem:** The docblock is longer than the function and repeats it.

```ts
// Before
/** Collapses whitespace for rule matching. */
export function normalizeSegment(text: string): string {
	return text.replace(/\s+/g, " ");
}

// After
export function normalizeSegment(text: string): string {
	return text.replace(/\s+/g, " ");
}
```

### 5. Types repeated in docs

**Watch for:** parameter and return types in a docblock that the signature already declares.

**Problem:** It states the same fact twice, and only the signature is checked. Where the language declares types, document meaning and leave the types to the signature. Keep docblock types only when they say more than the language can, such as the element type of an array.

```php
// Before
/**
 * @param int $id The ID.
 * @param bool $force Whether to force.
 * @return User The user.
 */
public function find(int $id, bool $force = false): User

// After
public function find(int $id, bool $force = false): User
```

```php
// Kept: the docblock types say more than `array`
/**
 * @param array<string, list<int>> $groups Member IDs by role name.
 * @return list<User>
 */
public function members(array $groups): array
```

### 6. Paragraph docblocks

**Watch for:** a docblock that walks through the implementation step by step, or runs past three or four lines on a function that is not a public entry point.

**Problem:** It describes how the function works, so it goes stale the first time the body changes. Describe what the function is for instead: its intent holds as long as the function does its job, whatever the steps are. Then add only the inputs, outputs, edge cases, errors and side effects a caller cannot guess.

```ts
// Before
/**
 * Reads the delimiter after `<<` at `start`. First it checks for a dash, which
 * means leading tabs are stripped. Then it skips spaces and tabs. Then it reads
 * the word, removing quotes and backslashes, and remembers whether any part was
 * quoted. Returns undefined when there is no delimiter.
 */

// After
/** The word that ends the here-document opened at `start`; `quoted` means its body is literal. Undefined when missing. */
```

The first version breaks when the parsing changes. The second stays true as long as the function finds the right delimiter.

### 7. History and change notes

**Watch for:** "now", "no longer", "used to", "was replaced", "previously", "new", "changed to".

**Problem:** The reader sees the current code. The diff belongs in commit messages and changelogs.

```ts
// Before
// Grants are now by folder; the old per-command grants were removed.

// After
(remove it; the commit message already says it)
```

### 8. Section banners

**Watch for:** `// ----- Paths -----`, `// ===== Helpers =====`, region markers around a few functions.

**Problem:** They drift as functions move and say nothing the function names do not. Remove them in code you are editing.

### 9. Commented-out code

**Problem:** Version control already keeps it. Remove it, unless the comment says when it comes back and why.

## Docblocks: when and how

A function earns a docblock when at least one of these holds:

- It is part of a public API, exported for other code to use.
- Its name and types leave a real question open: a unit, a range, what happens on empty input, what it throws, whether it mutates its arguments.
- It has a precondition or a side effect the caller has to know about.

A docblock that earns its place has one summary line with the function's intent: what it is for, in terms of the caller's problem. Leave the steps out, since they change with the body. After that it lists only what is not obvious: a parameter whose meaning the name does not carry, an edge case, an error.

It also lists the side effects the caller cannot see: events dispatched, jobs queued, mail sent, cache cleared, a transaction opened, a lazy load the caller must prevent. These are the consequences a reader otherwise digs out of the body, and often out of several other files.

It does not repeat types the signature declares. Two to four lines is the usual size. A longer one should be explaining something hard; if it retells the body, cut it.

```ts
/** Every path `arg` can resolve to: one if absolute, one per possible working directory otherwise. */
function argPaths(arg: PathArg, wd: WorkingDirs, project: string): ResolvedArg[]
```

```php
/**
 * Charges the order once; a repeated call returns the first charge.
 * Dispatches OrderPaid, which queues the receipt email and the stock update.
 *
 * @throws PaymentDeclined when the provider refuses the card.
 */
public function charge(Order $order): Charge
```

```php
/** Sum of the invoice lines. Lazy-loads `lines`: eager-load them when totaling many invoices. */
public function total(Invoice $invoice): Money
```

## When not to act

- A comment a linter or another tool requires: `// SAFETY:`, `// eslint-disable-next-line rule -- reason`, `@phpstan-ignore` with its reason.
- Licence headers and generated-code markers.
- Annotations that tools read, such as `@deprecated`, `@internal`, `@template`, and the `@property`, `@method` and `@mixin` blocks that describe magic properties and facades to the IDE and static analysis (Laravel IDE Helper and Larastan use them). They are types for tools, even when they look like redundant documentation.
- Attributes and decorators, which are code: `#[ObservedBy]`, `#[ScopedBy]`, `#[Override]`, `#[Deprecated]`, `#[Route]`.
- Comments in code the task does not touch, unless the task is a cleanup.
- A comment you cannot judge because you do not know the outside constraint it describes. Keep it, and mention it in your summary.
