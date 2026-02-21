# Feature Research

**Domain:** Language server grammar/parser fixes and completion enhancements
**Researched:** 2026-02-20
**Confidence:** HIGH (sourced from codebase archaeology, LSP 3.17 spec, Langium 4 internals, BBj grammar analysis)

---

## Milestone Scope

This file covers only **new features** for v3.9: parser bug fixes (#379, #380, #381, #382), grammar verb additions (EXIT int, SERIAL, ADDR refinement), and completion feature additions (static methods, .class property, constructor completion, deprecated indicators). All existing shipped features are out of scope.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features that BBj developers assume should work, based on what every other language server provides. Missing these makes the tool feel broken or incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **All valid BBj verbs parse without errors** | Users of any language server expect the grammar to cover the full language. `EXIT 0` is a standard BBj statement — it failing to parse causes cascading false errors. | LOW | `ExitWithNumberStatement` already has the token, but the grammar rule `kind='EXIT' \| RELEASE_NL \| RELEASE_NO_NL exitVal=Expression` has an operator-precedence ambiguity: the `\|` is parsed as "EXIT alone, or RELEASE with exitVal" not "EXIT or RELEASE, both with exitVal". Fix: restructure the rule so exitVal is optional on EXIT, or add a dedicated `ExitNumberStatement` alternative. |
| **SERIAL verb recognized in grammar** | BBj developers use SERIAL as a standard I/O control verb. Parse errors from an unknown keyword cascade through the entire statement. | LOW | SERIAL is currently absent from `bbj.langium`. The `KeywordStatement` rule handles many standalone verbs — SERIAL likely fits there, or needs its own rule if it takes arguments. Depends on actual BBj SERIAL syntax (BBj SERIAL sets the serial port — takes arguments). |
| **ADDR function recognized in grammar** | ADDR already exists in `AddrStatement` as a statement. But `ADDR()` used in expressions (e.g., `x = ADDR("prog")`) would need it as a `LibFunction` in `functions.ts`. The statement form is already in the grammar — the question is whether the function form also needs adding. | LOW | `AddrStatement` in grammar already covers statement use. Existing test at parser.test.ts:993 passes. The issue #377 may be about ADDR returning an expression value. Check if existing grammar handles `let x! = ADDR("prog")` form. |
| **`releaseVersion!` and similar keyword-prefixed identifiers parse correctly** | Variables like `releaseVersion!`, `stepXYZ!`, `addrOf!` must not trigger parse errors. This is the LONGER_ALT disambiguation pattern already established in v3.2 and v3.4. | LOW | #379: `releaseVersion!` — RELEASE is a keyword captured by `RELEASE_NL` / `RELEASE_NO_NL` custom tokens. These two tokens are added to the ID category (line 54-56 in bbj-token-builder.ts) but their `LONGER_ALT` points to just `id`, not `[idWithSuffix, id]`. The fix is the same as the v3.4 generic LONGER_ALT order fix — ensure RELEASE_NL and RELEASE_NO_NL have `LONGER_ALT = [idWithSuffix, id]` so `releaseVersion!` tokenizes as ID_WITH_SUFFIX. |
| **DECLARE statement valid anywhere in class scope** | BBj allows DECLARE at class level outside method bodies. The current grammar places DECLARE only in the `Statement` union, which is valid inside method bodies. Class-level DECLARE should also be accepted as a `ClassMember`. | MEDIUM | #380: Class grammar allows `members+=ClassMember` where `ClassMember = FieldDecl \| MethodDecl`. DECLARE generates a `VariableDecl` node — if the grammar doesn't include `VariableDecl` as a valid `ClassMember`, a class-level `declare auto BBjString name!` produces a parse error. Fix: extend `ClassMember` to include `VariableDecl`, or handle it in the `BbjClass` rule body. |
| **config.bbx files highlighted correctly** | TextMate grammar highlighting of the BBj configuration file. Users expect the editor to apply syntax coloring to the config file they interact with daily. | LOW | #381: `.bbx` files are now registered as BBj source in the language server (v3.1). But `config.bbx` is a config file format, not BBj source code. The fix is either: (a) add a separate grammar for `.bbx` config syntax, or (b) ensure the TextMate grammar correctly associates `config.bbx` vs program `.bbx` files. Likely: `config.bbx` is no longer highlighted because `.bbx` was merged into BBj language in v3.4 — it now gets BBj source grammar rather than the separate config grammar. Need to check TextMate grammar scoping. |
| **BUI/DWC startup succeeds with EM config set** | BUI/DWC run commands are the central feature for web-app developers. Failure to launch when an EM config path is set blocks a subset of users entirely. | LOW | #382: `web.bbj` line 69 checks `sscp! <> "--"` as a sentinel. The IntelliJ plugin's `BbjRunBuiAction` passes `classpath` directly (not `"--"` when empty). The `emUrl` field exists in `BbjSettings` but is never passed as an ARGV to `web.bbj`. If EM is at a non-default URL, `BBjAdminFactory.getBBjAdmin(token!)` connects to the configured EM host — but `web.bbj` doesn't accept the EM URL. The fix is either passing the EM URL to `web.bbj` as a new ARGV or using a BBjAdminFactory overload that accepts the URL. |

### Differentiators (Competitive Advantage)

Features that go beyond parser correctness and give BBj developers IDE-quality intelligence.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Static method completion on class references** | Typing `MyClass.` after a USE statement should propose static methods, matching what Java-interop provides for Java classes. Currently, `MemberCall` scope resolution returns instance members when the receiver is a `BbjClass`. | MEDIUM | #374: In `BbjScopeProvider.getScope()`, the `context.property === 'member' && isMemberCall()` branch calls `createBBjClassMemberScope()`. When the receiver is a `SymbolRef` whose symbol is a `BbjClass` (the class itself, not an instance), the scope should return only static members. The type inferer returns the `BbjClass` node when a `SymbolRef` resolves to a class. Requires: (1) detect when receiver resolves to a class (not an instance), (2) filter `createBBjClassMemberScope()` to `static` members only. `FieldDecl` and `MethodDecl` both have `static?: boolean` in the AST interface. |
| **`.class` property resolves on BBj class references** | `MyClass.class` is valid BBj — it returns a `java.lang.Class` object. Currently triggers "unresolvable member" warning. | LOW | #373: In the `MemberCall` scope resolution, when the receiver resolves to a `BbjClass` and the member name is `"class"`, return a synthetic `JavaClass` reference for `java.lang.Class`. The scope provider's `createBBjClassMemberScope()` result is used for member resolution — need to inject a synthetic `"class"` descriptor pointing to `java.lang.Class`. Pattern: similar to how `BBjAPI` is resolved via a synthetic document (`bbj-api.ts`). |
| **Constructor completion for `new ClassName()`** | After typing `new MyClass(`, the completion popup should show the constructor signature with parameter snippets. This is standard IDE behavior in Java/TypeScript. | MEDIUM | Constructor syntax is `ConstructorCall: 'new' klass=QualifiedClass '(' args... ')'`. The `BBjCompletionProvider` currently handles `createReferenceCompletionItem` for method references but not constructors. Need: (1) detect when cursor is inside a `ConstructorCall` node's arg position, (2) look up the `BbjClass.methods` to find methods named same as the class (BBj convention for constructors), or look up the `JavaClass` constructor list from `ClassDoc.constructors`. Java constructors are in `java-javadoc.ts`'s `ClassDoc.constructors?: MethodDoc[]` — already defined but marked TODO in `getDocumentation()`. |
| **Deprecated method visual indicator in completion** | Methods marked `@deprecated` in Javadoc should show with strikethrough decoration in the completion list. VS Code supports `CompletionItem.tags: [CompletionItemTag.Deprecated]`. IntelliJ (via LSP4IJ) also renders this. | LOW | Requires: (1) detect `@deprecated` in Javadoc text during class resolution in `java-interop.ts`, (2) set `tags: [1]` (`CompletionItemTag.Deprecated`) on the `CompletionItem` in `bbj-completion-provider.ts`. The `JavaMethod.docu.javadoc` string already contains the raw Javadoc — check if it contains `@deprecated` or `{@deprecated}`. The `FunctionNodeDescription` used in `createReferenceCompletionItem` would need a `deprecated?: boolean` field. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Showing all methods (static + instance) on `MyClass.`** | "Just give me everything" | Pollutes completion with instance methods that require an object, causing confusion and incorrect code. Java IDEs and TypeScript strictly separate static vs instance completion contexts. | Filter to static-only when receiver is a class reference, instance-only when receiver is an instance. |
| **DECLARE as a BBj keyword in completion** | Users see it frequently | `declare` is Langium's keyword but is already suggested via the keyword completion pipeline. Making it a `LibMember` would duplicate it. | The keyword completion in `completionForKeyword()` already handles it — no duplication needed. |
| **Blocking constructor resolution on missing Java interop** | "Show constructors even when interop is down" | Constructor metadata (parameter names/types) comes from `java-interop`. Without it, showing partial/wrong signatures misleads users more than showing nothing. | Show basic `new ClassName()` label without parameter details when interop is unavailable; enhance with signature when available. |
| **Making `.class` return `BBjClass` type** | Seems natural | In BBj/Java, `.class` always returns `java.lang.Class<T>` — it is a Java reflection type. Treating it as a `BBjClass` would cause false type errors when calling Java `Class<?>` methods on the result. | Resolve `.class` to `java.lang.Class` from the Java interop classpath. |

---

## Feature Dependencies

```
[EXIT int fix]
    └──requires──> [RELEASE_NL/RELEASE_NO_NL token LONGER_ALT order fix]

[releaseVersion! fix (#379)]
    └──requires──> [RELEASE_NL/RELEASE_NO_NL LONGER_ALT = [idWithSuffix, id]]
    └──shares mechanism with──> [EXIT int fix] (same root cause: RELEASE tokens)

[SERIAL verb (#375)]
    └──requires──> [grammar rule addition in bbj.langium]
    └──requires──> [token generation via langium-cli]

[ADDR expression form (#377)]
    └──requires──> [investigation: is statement form sufficient, or does expression form need LibFunction entry?]

[DECLARE in class (#380)]
    └──requires──> [ClassMember grammar extended or BbjClass body rule modified]

[config.bbx highlighting (#381)]
    └──requires──> [TextMate grammar investigation: how .bbx was scoped before v3.4 merge]
    └──conflicts with──> [.bbx as BBj source (existing v3.1 decision)]

[EM Config "--" fix (#382)]
    └──requires──> [understanding web.bbj ARGV contract]
    └──requires──> [passing emUrl or fixing BBjAdminFactory connection target]

[Static method completion (#374)]
    └──requires──> [type inferer correctly identifies class-reference vs instance receivers]
    └──requires──> [createBBjClassMemberScope() accepts static-only filter flag]
    └──enhances──> [existing MemberCall scope resolution]

[.class property (#373)]
    └──requires──> [java.lang.Class available from java-interop classpath]
    └──requires──> [MemberCall scope resolution special-cases "class" member name]
    └──enhances──> [static method completion] (both triggered by class-reference receiver)

[Constructor completion]
    └──requires──> [BBjClass constructor lookup OR Java ClassDoc.constructors population]
    └──requires──> [CompletionContext detection of ConstructorCall argument position]
    └──depends on──> [java-javadoc.ts ClassDoc.constructors field already defined]

[Deprecated method indicator]
    └──requires──> [Javadoc text scanning for @deprecated in java-interop.ts or completion provider]
    └──requires──> [FunctionNodeDescription.deprecated boolean field]
    └──requires──> [CompletionItem.tags = [CompletionItemTag.Deprecated] in createReferenceCompletionItem]
    └──depends on──> [existing docu.javadoc string population (v3.8 shipped)]
```

### Dependency Notes

- **releaseVersion! fix and EXIT int fix share the same root cause**: RELEASE_NL/RELEASE_NO_NL tokens currently have `LONGER_ALT = id` (not `[idWithSuffix, id]`). Fixing the token order resolves both issues at once. This is consistent with the v3.4 fix for all keyword tokens — RELEASE tokens were handled separately and missed the upgrade.
- **Static method completion and .class property share context detection**: Both require knowing "the receiver is a class, not an instance." Once that detection is in place in `BbjScopeProvider`, both features can share it.
- **Deprecated indicator depends on v3.8's javadoc enrichment**: The `docu.javadoc` field is now populated during class resolution (v3.8 shipped). Scanning that string for `@deprecated` is a lightweight addition.
- **Constructor completion has a split path**: BBj classes (defined in `.bbj` files) don't have an explicit constructor syntax — methods with the same name as the class are used. Java classes have `ClassDoc.constructors` in the javadoc provider, already declared but not populated in `getDocumentation()`. Both paths need separate handling.
- **DECLARE in class (#380) requires grammar change**: This triggers a `langium-cli generate` step to regenerate `generated/ast.ts` and `generated/grammar.ts`. The grammar change must be coordinated with any AST type usage in validators and scope providers.

---

## MVP Definition

### Launch With (v3.9 milestone)

Bug fixes (blocking correctness):
- [x] **EXIT int optional parameter** (#376) — fix `ExitWithNumberStatement` grammar so `EXIT 0` and bare `EXIT` both parse
- [x] **releaseVersion! identifier** (#379) — fix RELEASE_NL/RELEASE_NO_NL LONGER_ALT to `[idWithSuffix, id]`
- [x] **DECLARE in class** (#380) — extend `ClassMember` to accept `VariableDecl`
- [x] **EM Config "--" DWC/BUI failure** (#382) — fix the emUrl/config passing contract in web.bbj or plugin
- [x] **config.bbx highlighting** (#381) — restore syntax highlighting for BBj config files

Grammar additions:
- [x] **SERIAL verb** (#375) — add to grammar (standalone or with arguments per BBj docs)
- [x] **ADDR function form** (#377) — confirm if statement form suffices or expression form needs LibFunction

Completion features:
- [x] **Static method completion on class references** (#374) — filter to static members when receiver is a class
- [x] **.class property resolution** (#373) — resolve to java.lang.Class, no false "unresolvable member" warning
- [x] **Constructor completion** — snippet completion for `new ClassName(...)` with parameter names
- [x] **Deprecated method indicator** — `CompletionItemTag.Deprecated` on items with `@deprecated` in Javadoc

### Add After Validation (v3.9.x)

- [ ] **Constructor completion with overloads** — show all constructor overloads as separate completion items (if a class has multiple `new` forms)
- [ ] **Static field completion on class references** — same mechanism as static methods, but for FIELD members marked static

### Future Consideration (v4+)

- [ ] **`instanceOf` type narrowing** — static completion on cast results
- [ ] **Full class member visibility enforcement** — warn when calling private static method from outside class

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| releaseVersion! fix (#379) | HIGH (parse errors on valid code) | LOW (LONGER_ALT tweak) | P1 |
| EXIT int fix (#376) | HIGH (parse errors on valid code) | LOW (grammar tweak) | P1 |
| DECLARE in class (#380) | HIGH (parse errors on valid code) | LOW-MEDIUM (grammar + AST) | P1 |
| EM Config "--" fix (#382) | HIGH (BUI/DWC broken for users with EM config) | LOW-MEDIUM (web.bbj + plugin) | P1 |
| config.bbx highlighting (#381) | MEDIUM (visual regression) | LOW (TextMate grammar) | P1 |
| SERIAL verb (#375) | MEDIUM (parse error on valid verb) | LOW (grammar addition) | P1 |
| ADDR function form (#377) | MEDIUM (depends on investigation) | LOW (LibFunction entry if needed) | P1 |
| Static method completion (#374) | HIGH (core OOP workflow) | MEDIUM (scope filter + type detection) | P1 |
| .class property (#373) | MEDIUM (false warning elimination) | LOW-MEDIUM (synthetic member injection) | P2 |
| Constructor completion | HIGH (fundamental OOP pattern) | MEDIUM (completion context detection) | P2 |
| Deprecated method indicator | MEDIUM (noise reduction in completion) | LOW (tag field + Javadoc scan) | P2 |

**Priority key:**
- P1: Must have — either blocking bugs or high-value quick wins
- P2: Should have — complete the completion feature story
- P3: Nice to have — defer

---

## Behavioral Specification by Feature

### EXIT int Optional Parameter (#376)

**Current behavior:** `EXIT 0` produces a parse error. `EXIT` alone works.

**Root cause:** `ExitWithNumberStatement: kind='EXIT' | RELEASE_NL | RELEASE_NO_NL exitVal=Expression` — the Langium grammar alternation `|` without grouping means this parses as `(kind='EXIT') | (RELEASE_NL exitVal=Expression) | (RELEASE_NO_NL exitVal=Expression)`. EXIT gets no exitVal.

**Expected behavior:** Both `EXIT` (no arg) and `EXIT 0` / `EXIT expr` parse without errors. The integer is optional.

**Fix pattern:**
```langium
ExitStatement:
    kind='EXIT' exitVal=Expression?
;
ReleaseStatement:
    kind=(RELEASE_NL | RELEASE_NO_NL) exitVal=Expression
;
```
Or keep combined but with correct grouping:
```langium
ExitWithNumberStatement:
    (kind='EXIT' exitVal=Expression?) | ((RELEASE_NL | RELEASE_NO_NL) exitVal=Expression)
;
```

---

### releaseVersion! Identifier Fix (#379)

**Current behavior:** `releaseVersion!` produces a lexer error because `RELEASE_NO_NL` matches "RELEASE" first, leaving "Version!" unmatched.

**Root cause:** In `bbj-token-builder.ts`, lines 54-56:
```typescript
releaseNl.LONGER_ALT = id;      // should be [idWithSuffix, id]
releaseNoNl.LONGER_ALT = id;    // should be [idWithSuffix, id]
```
The LONGER_ALT should prefer `idWithSuffix` first (matching `releaseVersion!`) before falling back to `id`.

**Expected behavior:** `releaseVersion!` tokenizes as `ID_WITH_SUFFIX`. No parse error.

---

### DECLARE in Class (#380)

**Current behavior:** `declare auto BBjString name!` written at class level (not inside a METHOD) triggers a parse error.

**Root cause:** `ClassMember = FieldDecl | MethodDecl` — `VariableDecl` is not listed as a valid class member.

**Expected behavior:** `declare` statements at class scope parse correctly. BBj allows class-level declares for type annotations.

**Fix pattern:**
```langium
ClassMember returns ClassMember:
    FieldDecl | MethodDecl | VariableDecl
;
```
Note: this requires `VariableDecl` to be a subtype of `ClassMember` in the interface hierarchy, or `ClassMember` redefined as a union type. AST regeneration required.

---

### EM Config "--" Fix (#382)

**Current behavior:** With an EM config path set in settings, BUI/DWC launch fails.

**Root cause (hypothesis):** `web.bbj` receives the config path as ARGV(9). The IntelliJ `BbjRunBuiAction` passes `configPath` at argument position 9. However, `BBjAdminFactory.getBBjAdmin(token!)` uses the default EM connection (localhost:8888). If the user has EM at a different URL (stored in `emUrl` setting), the `getBBjAdmin` call fails. The `emUrl` field exists in `BbjSettings.State` but is never passed to `web.bbj`.

**Expected behavior:** When EM URL is configured, web.bbj uses it for the `BBjAdminFactory` connection. Or the IntelliJ plugin uses the correct BBjAdminFactory API that accepts the URL.

**Fix pattern:** Pass `emUrl` as ARGV(10) to web.bbj, then use it in the BBjAdminFactory call with the URL-accepting overload.

---

### Static Method Completion (#374)

**Current behavior:** `MyClass.` after a USE statement shows no completions or instance-only completions.

**Expected behavior:** `MyClass.someStaticMethod()` proposes static methods and static fields with snippets. Instance methods not shown.

**Mechanism:**
1. In `BbjTypeInferer.getTypeInternal()`, when a `SymbolRef` resolves to a `BbjClass` node (not an instance), `getType()` returns that class.
2. In `BbjScopeProvider.getScope()`, the `isMemberCall()` branch checks `receiverType`. When `isBbjClass(receiverType)`, current code calls `createBBjClassMemberScope(receiverType)` without filtering.
3. Fix: add a `staticOnly` flag. When receiver is a class reference (not an instance), call `createBBjClassMemberScope(receiverType, staticOnly=true)`.
4. In `createBBjClassMemberScope()`, already has a `methodsOnly` flag pattern — add `staticOnly` alongside it.
5. `FieldDecl.static` and `MethodDecl.static` are boolean fields in the AST.

---

### .class Property (#373)

**Current behavior:** `MyClass.class` triggers an "unresolvable member" warning.

**Expected behavior:** `MyClass.class` resolves to `java.lang.Class`, no warning, and hover shows `Class<MyClass>`.

**Mechanism:**
1. In `BbjScopeProvider.getScope()`, when `isMemberCall()` and `isBbjClass(receiverType)`, detect if `context.container.member.$refText === 'class'`.
2. If so, look up `java.lang.Class` from `javaInterop.getResolvedClass('java.lang.Class')` and return a scope containing it.
3. Or: add a synthetic `"class"` entry to the class member scope that resolves to the `java.lang.Class` JavaClass node.

---

### Constructor Completion

**Current behavior:** `new MyClass(` triggers no parameter hints or completion.

**Expected behavior:** Completion popup shows `MyClass(param1, param2)` snippet(s) with documentation.

**Mechanism (BBj classes):**
- BBj classes don't declare explicit constructors. The convention is methods with the same name as the class. Look up `klass.members.filter(isMethodDecl).filter(m => m.name === klass.name)` for constructor candidates.
- If found, the completion provider should detect `ConstructorCall` context and provide these methods as constructor options.

**Mechanism (Java classes):**
- `ClassDoc.constructors?: MethodDoc[]` already defined in `java-javadoc.ts` but the `TODO handle constructors` comment in `getDocumentation()` means constructors are not currently returned.
- Fix: handle `'JavaClass'` (for constructor lookup) and populate the constructor completions from `ClassDoc.constructors`.

---

### Deprecated Method Indicator

**Current behavior:** Deprecated Java methods appear in completion with no visual distinction.

**Expected behavior:** Methods with `@deprecated` in Javadoc appear with strikethrough decoration in the completion popup.

**LSP mechanism:** `CompletionItem.tags: [CompletionItemTag.Deprecated]` (value `1`) — supported by VS Code and IntelliJ/LSP4IJ.

**Implementation:**
1. During class resolution in `java-interop.ts`, when populating `method.docu.javadoc`, check if the string contains `"@deprecated"` (case-insensitive).
2. Add `deprecated?: boolean` to `FunctionNodeDescription` in `bbj-nodedescription-provider.ts`.
3. In `BBjCompletionProvider.createReferenceCompletionItem()`, when `isFunctionNodeDescription(nodeDescription)` and `nodeDescription.deprecated === true`, set `superImpl.tags = [1]`.

---

## Research Findings by Feature Category

### Grammar Fixes: Lessons from Prior Milestones

**Finding (HIGH confidence — direct codebase analysis):**

The BBj grammar has a recurring pattern of disambiguation issues from BBj's non-reserved keywords. Solutions established in v3.0, v3.2, v3.4:

1. **LONGER_ALT pattern**: Every keyword token that can appear as an identifier prefix must have `LONGER_ALT = [idWithSuffix, id]` (not just `id`). The v3.4 fix applied this to all keyword tokens via the loop in `bbj-token-builder.ts` lines 39-49. However, `RELEASE_NL` and `RELEASE_NO_NL` are handled separately at lines 54-56 and were not updated to use `[idWithSuffix, id]`. This is the root cause of #379.

2. **Grammar alternation grouping**: Langium grammar `A | B C` means `(A) | (B C)`. To make C optional for A: `(A C?) | (B C)`. This is the EXIT fix pattern.

3. **langium-cli regenerate requirement**: Any grammar change requires running `langium generate` to regenerate `generated/ast.ts`. This means grammar fixes require a build step before tests can run — worth noting in phase planning.

### Completion: How LSP Handles Static vs Instance Members

**Finding (HIGH confidence — LSP 3.17 spec, TypeScript LS behavior, Eclipse JDT behavior):**

Standard behavior in typed language servers:
- When the receiver of a member access is a **class name** (not an instance), only static members are proposed.
- When the receiver is an **instance variable**, only instance members are proposed.
- This requires the type system to distinguish "a reference to the class itself" vs "an instance of the class."

In the BBj type inferer:
- `isClass(reference)` in `getTypeInternal()` already returns the class node itself when a `SymbolRef` resolves to a class.
- This is the signal: if `isBbjClass(getType(receiver))` AND the receiver `SymbolRef` is a class (not an instance variable declared as that class), return only static members.
- The current code returns all members regardless — the distinction exists in the AST but is not used in scope filtering.

### .class Property: Java Reflection Pattern

**Finding (HIGH confidence — Java language spec, BBj documentation pattern):**

`.class` is a Java/BBj pseudo-property available on any type reference. It returns `java.lang.Class<T>`. This is not a method call — it is a special syntax that the compiler handles at the grammar/type-system level.

In LSP terms: when `member.$refText === 'class'` and the receiver is a class reference, the member should resolve to a synthetic element typed as `java.lang.Class`. This cannot be found in the class's member list — it must be injected by the scope provider as a special case, similar to how `length` is a special property on arrays in TypeScript.

**Implementation pattern from similar LSs:** TypeScript's `typeof MyClass` and `.constructor` properties are handled as special cases in the type checker. The simplest BBj approach: inject a synthetic `AstNodeDescription` for `"class"` pointing to the `java.lang.Class` JavaClass node (if resolved by java-interop) when the receiver is a BBj or Java class.

### Constructor Completion: LSP Convention

**Finding (MEDIUM confidence — LSP 3.17 spec, TypeScript/Java completion behavior):**

Constructor completion is typically triggered after `new ClassName(`. The LSP `CompletionParams` will have a context with the cursor inside the `ConstructorCall` argument list. The completion provider needs to:
1. Detect that the cursor is inside a `ConstructorCall` node (via AST walk from cursor position).
2. Look up constructor signatures for that class.
3. Return completions with `insertTextFormat: 2` (snippet) and `insertText: "MyClass(${1:param1}, ${2:param2})"`.

The challenge in Langium: the `DefaultCompletionProvider`'s grammar follower traversal drives what completions are offered. Inside `ConstructorCall`'s argument position, the grammar allows `Expression` — so all variables/symbols are offered (correct). But constructor-specific signature help typically comes from `signatureHelp`, not `completion`. The completion popup approach is more of a "show the constructor options before the user types the args" pattern.

**Recommendation:** Implement constructor completion as an enhancement to `SignatureHelpProvider` (already exists in `bbj-signature-help-provider.ts`) rather than as completion items. This is how TypeScript works: typing `new MyClass(` triggers signature help, not a completion popup.

### Deprecated Indicators: LSP Support

**Finding (HIGH confidence — LSP 3.17 spec, VS Code and IntelliJ/LSP4IJ behavior):**

LSP 3.17 defines `CompletionItemTag.Deprecated = 1`. When set in `CompletionItem.tags`, VS Code renders the label with strikethrough. LSP4IJ (IntelliJ's LSP bridge) also supports this tag — confirmed in LSP4IJ changelog for recent versions.

The check is simple: `docu.javadoc.toLowerCase().includes('@deprecated')`. The Javadoc `@deprecated` tag is standard Java documentation convention.

---

## Sources

- Codebase analysis: `bbj-vscode/src/language/bbj.langium` (grammar rules for EXIT, DECLARE, ADDR)
- Codebase analysis: `bbj-vscode/src/language/bbj-token-builder.ts` (RELEASE_NL LONGER_ALT gap at lines 54-56)
- Codebase analysis: `bbj-vscode/src/language/bbj-scope.ts` (MemberCall scope resolution, static field present in AST)
- Codebase analysis: `bbj-vscode/src/language/bbj-completion-provider.ts` (createReferenceCompletionItem, existing tag infrastructure)
- Codebase analysis: `bbj-vscode/src/language/java-javadoc.ts` (ClassDoc.constructors defined, TODO comment)
- Codebase analysis: `bbj-vscode/src/language/java-types.langium` (JavaMethod, JavaField — no `static` field present)
- Codebase analysis: `bbj-vscode/tools/web.bbj` (ARGV contract, emUrl not passed)
- Codebase analysis: `bbj-intellij/src/main/java/com/basis/bbj/intellij/actions/BbjRunBuiAction.java` (emUrl stored but not used in command)
- Codebase analysis: `bbj-vscode/test/parser.test.ts` line 993 (ADDR test already passing — statement form works)
- LSP 3.17 Specification: CompletionItemTag, CompletionItem.tags — https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/
- Langium 4.1.3 grammar documentation: LONGER_ALT disambiguation
- v3.4 commit context: generic LONGER_ALT fix that missed RELEASE tokens
- v3.0 commit context: LONGER_ALT array [id, idWithSuffix] disambiguation origin

---

*Feature research for: BBj Language Server v3.9 — Grammar Fixes, Parser Bug Fixes, Completion Enhancements*
*Researched: 2026-02-20*
