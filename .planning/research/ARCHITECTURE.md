# Architecture Research

**Domain:** Langium-based BBj Language Server — grammar additions, parser bug fixes, run command fixes, highlighting fixes, completion enhancements
**Researched:** 2026-02-20
**Confidence:** HIGH (direct codebase inspection, no speculative claims)

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          IDE Layer                                   │
│  ┌──────────────────────┐         ┌────────────────────────────┐    │
│  │   bbj-vscode/        │         │   bbj-intellij/            │    │
│  │   extension.ts       │         │   BbjRunActionBase.java    │    │
│  │   Commands.cjs       │         │   (BUI/DWC/GUI actions)    │    │
│  │   (BUI/DWC/GUI run)  │         │   LSP4IJ bridge            │    │
│  └──────────┬───────────┘         └─────────────┬──────────────┘    │
│             │ stdio / LSP                        │ stdio / LSP       │
└─────────────┼──────────────────────────────────-┼───────────────────┘
              │                                    │ (same binary)
┌─────────────▼────────────────────────────────────▼──────────────────┐
│                    Language Server (Node.js)                         │
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ bbj-module  │  │ bbj-lexer   │  │ bbj-token-  │                 │
│  │ (DI wiring) │  │ (line-cont) │  │ builder     │                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │                │                │                         │
│  ┌──────▼──────────────────────────────────▼──────────────────┐    │
│  │                    Langium Core Pipeline                     │    │
│  │  Grammar (bbj.langium) -> Parser -> AST                     │    │
│  └──────────────────────────────┬───────────────────────────────┘   │
│                                 │ AST                               │
│  ┌────────────────┐  ┌──────────▼──────────┐  ┌─────────────────┐  │
│  │ bbj-scope      │  │ bbj-scope-local     │  │ bbj-linker      │  │
│  │ (global scope/ │  │ (LocalSymbols /     │  │ (cross-ref      │  │
│  │  member scope) │  │  export symbols)    │  │  resolution)    │  │
│  └────────┬───────┘  └─────────────────────┘  └────────┬────────┘  │
│           │                                             │            │
│  ┌────────▼───────────────────────────────────────────-▼──────┐    │
│  │ bbj-type-inferer    bbj-validator    bbj-completion-provider │    │
│  │ (Expression->Class) (semantic rules) (LSP completion items) │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────┐  ┌─────────────────────────────────────┐  │
│  │ bbj-document-       │  │ bbj-ws-manager                      │  │
│  │ builder (BBjCPL     │  │ (workspace init, settings,          │  │
│  │  integration)       │  │  classpath, java-interop config)    │  │
│  └─────────────────────┘  └─────────────────────────────────────┘  │
└────────────────────────────────────────┬────────────────────────────┘
                                         │ JSON-RPC / TCP socket
┌────────────────────────────────────────▼────────────────────────────┐
│                   java-interop (Java process)                        │
│   InteropService: getClassInfo, getClassInfos, loadClasspath,        │
│   getTopLevelPackages -> JavaClass/JavaMethod/JavaField metadata     │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Key File |
|-----------|----------------|----------|
| `bbj.langium` | Grammar defining all BBj syntax: statements, expressions, class/method decls, terminals | `bbj-vscode/src/language/bbj.langium` |
| `java-types.langium` | Type hierarchy for Java interop: JavaClass, JavaMethod, JavaField, DocumentationInfo | `bbj-vscode/src/language/java-types.langium` |
| `BbjLexer` | Pre-processing: collapses `:` line-continuation before tokenization | `bbj-lexer.ts` |
| `BBjTokenBuilder` | Chevrotain token assembly: splices synthetic tokens (RELEASE_NL, RPAREN_NL, etc.), adds LONGER_ALT `[idWithSuffix, id]` on all uppercase keywords to prevent camel-case identifier breakage | `bbj-token-builder.ts` |
| `BbjScopeComputation` | Per-document local symbols: variables, fields, DEF FN params, class members, accessor descriptors, USE-resolved Java classes | `bbj-scope-local.ts` |
| `BbjScopeProvider` | Global + member scope: resolves MemberCall receiver type, Java package children, BBj class inheritance chain, named parameter scope | `bbj-scope.ts` |
| `BBjTypeInferer` | Expression to Class mapping: SymbolRef, ConstructorCall, MemberCall, CastExpression, StringLiteral to JavaClass | `bbj-type-inferer.ts` |
| `BBjValidator` | Semantic validation checks registered per AST node type | `bbj-validator.ts` + `validations/` |
| `BBjDocumentValidator` | Diagnostic hierarchy: BBjCPL > Parse > Semantic > Warning; cascading suppression and maxErrors cap | `bbj-document-validator.ts` |
| `BBjCompletionProvider` | LSP completion: `#` trigger for class fields, Java method snippets with Javadoc, symbolic labels, keywords | `bbj-completion-provider.ts` |
| `JavaInteropService` | Java class metadata: async resolution with mutex, depth limit, timeout, package hierarchy, Javadoc enrichment | `java-interop.ts` |
| `BBjWorkspaceManager` | Workspace init: settings parsing (home, classpath, interopHost, interopPort, configPath, diagnostic settings), library loading, BBjAPI synthetic doc | `bbj-ws-manager.ts` |
| `BBjDocumentBuilder` | Extends Langium build pipeline: BBjCPL compiler integration with 500ms debounce | `bbj-document-builder.ts` |
| `bbj-module.ts` | Langium DI wiring: BBjModule + BBjSharedModule, `createBBjServices()` | `bbj-module.ts` |
| TextMate grammars | Syntax highlighting: `bbj.tmLanguage.json` (BBj source), `bbx.tmLanguage.json` (BBx config files) | `syntaxes/` |
| `Commands.cjs` | VS Code run commands: `runWeb()` for BUI/DWC, `run()` for GUI — calls `web.bbj` with EM token and configPath | `src/Commands/Commands.cjs` |
| `web.bbj` | BBj runner: reads ARGV params (client, name, programme, wd, user, pass, classpath, token, configPath), registers app in EM, launches URL | `tools/web.bbj` |
| `BbjRunActionBase.java` | IntelliJ run actions: shared auto-save, validation, process management, stderr logging, `getConfigPath()`/`getConfigPathArg()` | `bbj-intellij/.../actions/BbjRunActionBase.java` |

---

## Integration Points for v3.9 Features

### A. Grammar Verb Additions (EXIT int, SERIAL, ADDR)

**Files touched:** `bbj.langium`, `generated/ast.ts` (auto-generated)

**Existing pattern for EXIT verb:**
```
ExitWithNumberStatement:
    kind='EXIT' | RELEASE_NL | RELEASE_NO_NL exitVal=Expression
;
```
Currently `EXIT` alone has no optional integer parameter. Adding `EXIT int` means making `exitVal` optional when `EXIT` is the keyword (as opposed to RELEASE). The grammar rule needs an optional expression after `kind='EXIT'`.

**Integration:**
- Grammar change only — no validator, scope, or completion changes needed
- `generated/ast.ts` and `generated/grammar.ts` are auto-regenerated by `langium generate`
- Test: add parser test in `bbj-vscode/test/parser.test.ts`

**SERIAL verb:** New `SingleStatement` alternative. Add a `SerialStatement` rule body and reference it in `SingleStatement`.

**ADDR function:** Exists as `AddrStatement` (`'ADDR' fileid=StringLiteral Err?`). Verify whether the issue is the rule's parameter type (StringLiteral vs. Expression) or a tokenizer conflict with ADDR appearing in other positions.

### B. Parser Bug: `releaseVersion!` Flagged as Error (#379)

**Root cause (HIGH confidence, verified by direct inspection):**

In `bbj-token-builder.ts`:
```typescript
releaseNl.LONGER_ALT = id;          // only id, NOT [idWithSuffix, id]
releaseNoNl.LONGER_ALT = id;
```

All other uppercase keyword tokens processed in the loop get:
```typescript
keywordToken.LONGER_ALT = [idWithSuffix, id];
```

When Chevrotain sees `releaseVersion!`, the RELEASE_NO_NL token matches "RELEASE" via its regex, then checks LONGER_ALT for a longer match. LONGER_ALT only includes `id` (matches `releaseVersion` without the `!`), not `idWithSuffix` (which would match `releaseVersion!`). So the tokenizer uses the ID token for `releaseVersion` and leaves `!` as an orphaned prefix operator, causing a parse error.

**Fix (2 lines in `bbj-token-builder.ts`):**
```typescript
releaseNl.LONGER_ALT = [idWithSuffix, id];
releaseNoNl.LONGER_ALT = [idWithSuffix, id];
```

**Files touched:** `bbj-token-builder.ts` only.

### C. Parser Bug: DECLARE Outside Methods in Class (#380)

**Root cause:** The grammar `ClassDecl` only allows `ClassMember` (FieldDecl | MethodDecl) or `Comments` between `CLASS` and `CLASSEND`. A `declare` statement at class level (not inside a method) does not match any of these alternatives, causing a parse error.

```
ClassDecl returns BbjClass:
    'CLASS' Visibility Static name=ValidName ...
    (members+=ClassMember | Comments)*
    'CLASSEND'
;
ClassMember returns ClassMember:
    FieldDecl | MethodDecl      // VariableDecl is not here
;
```

**Fix:** Add a grammar alternative in `ClassDecl` to accept (and silently discard) a `VariableDecl` at class scope, since BBj runtime ignores class-level DECLARE statements.

```
// Option: extend ClassMember to include VariableDecl at class scope
ClassMember returns ClassMember:
    FieldDecl | MethodDecl | ClassLevelDeclare
;
ClassLevelDeclare:
    {infer ClassLevelDeclare} 'declare' auto?='auto'? type=QualifiedClass (array?='[' ']')? name=FeatureName
;
```

Or, simpler: make `VariableDecl` a valid `ClassMember` alternative and don't export it to the class member scope.

**Files touched:** `bbj.langium` (grammar alternative in `ClassDecl`).

### D. Run Command Bug: EM Config "--" Causing DWC/BUI Failure (#382)

**Root cause (verified by inspecting `web.bbj`):**

In `tools/web.bbj` line 69:
```bbj
if sscp! <> null() and sscp! > "" and sscp! <> "--" then
    app!.setString(app!.CLASSPATH, sscp!)
fi
```

The classpath parameter is guarded against `"--"`. But `configFile!` (ARGV(9)) has no equivalent guard. If the EM configuration passes `"--"` as the configPath (e.g. because the configPath setting is not set but the parameter position is used as an EM config separator), it gets passed to `app!.setString(app!.CONFIG_FILE, "--")`, which causes the EM to use `"--"` as a literal config file path and fail to launch DWC/BUI.

**Fix:** Add the same `"--"` guard for `configFile!` in `web.bbj`:
```bbj
if (configFile! <> null() and configFile! > "" and configFile! <> "--") then
    app!.setString(app!.CONFIG_FILE, configFile!)
else
    app!.setString(app!.CONFIG_FILE, BBjAPI().getConfig().getConfigFileName())
fi
```

**Files touched:** `tools/web.bbj` (primary fix). No IntelliJ change needed if the web.bbj guard is sufficient.

### E. Highlighting Bug: config.bbx No Longer Highlighted (#381)

**Root cause (verified by inspecting `package.json`):**

`package.json` registers one grammar entry:
```json
"grammars": [
  {
    "language": "bbj",
    "scopeName": "source.bbj",
    "path": "./syntaxes/bbj.tmLanguage.json"
  }
]
```

The `bbx.tmLanguage.json` file exists in `syntaxes/` with `scopeName: "source.bbx"` but is NOT registered. The `bbj` language id covers `.bbx` extension (file type registration), but uses only `bbj.tmLanguage.json` for highlighting. Since `config.bbx` is a BBx-format configuration file (not BBj source), it should use `bbx.tmLanguage.json` grammar.

**Fix:** Add a second grammar entry in `package.json`:
```json
{
  "language": "bbj",
  "scopeName": "source.bbx",
  "path": "./syntaxes/bbx.tmLanguage.json"
}
```

Or, if `.bbx` files need to use the BBx grammar instead of the BBj grammar: use a filename-pattern override. The simplest fix for `config.bbx` specifically is to associate `.bbx` files with the `bbx` language id and register `bbx.tmLanguage.json` for it. But since `.bbx` files are currently treated as BBj source (they run as BBj programs), the best approach is to register `bbx.tmLanguage.json` as an embedded grammar for the `bbj` scope.

**Files touched:** `bbj-vscode/package.json`.

### F. Static Method Completion on Class References via USE (#374)

**Current behavior analysis (HIGH confidence):**

`BbjScopeProvider.getScope()` for `'member'` property:
```typescript
if (context.property === 'member' && isMemberCall(context.container)) {
    const receiverType = this.typeInferer.getType(receiver);
    if (isBbjClass(receiverType)) {
        return new StreamScopeWithPredicate(
            this.createBBjClassMemberScope(receiverType).getAllElements(),
            super.getScope(context)
        );
    }
}
```

When user writes `MyClass.someMethod()`, `typeInferer.getType(SymbolRef { symbol -> BbjClass })` returns the `BbjClass` node (via `isClass(reference)` branch in type inferer). The scope includes ALL members — both static and instance. The gap: instance members should NOT be offered when completing on a class reference.

**Fix:** Detect whether the receiver resolves to a class reference (not an instance) and return only static members:
```typescript
if (isBbjClass(receiverType)) {
    // receiverType is the class itself (from SymbolRef resolving to BbjClass)
    // Check if receiver is a direct class reference (not an object instance)
    const isClassRef = isSymbolRef(receiver) && isClass(receiver.symbol.ref);
    if (isClassRef) {
        // Return only static members for class-reference access
        return this.createBBjClassStaticMemberScope(receiverType);
    }
    return new StreamScopeWithPredicate(
        this.createBBjClassMemberScope(receiverType).getAllElements(),
        super.getScope(context)
    );
}
```

Add `createBBjClassStaticMemberScope()` that filters to `member.static === true`.

**Files touched:** `bbj-scope.ts` (scope provider), possibly `bbj-completion-provider.ts` (for consistent completion item kind).

### G. `.class` Property Resolution (#373)

**Current behavior:** `MyClass.class` parses as a `MemberCall` with `member` referencing the name `class`. However, `class` is a BBj keyword — it tokenizes as the `CLASS` keyword token, not as an `ID` or `FeatureName`. The `MemberCall` grammar:
```
MemberCall: PrimaryExpression (... '.' member=[NamedElement:FeatureName] ...)
```
`FeatureName` returns ID | ID_WITH_SUFFIX. The keyword `CLASS` has `CATEGORIES = [id]` (from token builder), so it can tokenize as ID via category. But whether it satisfies `FeatureName` after `.` is ambiguous.

**Fix approach:** In `BBjTypeInferer.getTypeInternal()`, handle the `MemberCall` pattern where `member.$refText === 'class'` as a special case returning `java.lang.Class`. No grammar change needed if the tokenizer already allows `CLASS` as an ID — just add a type inferer branch:

```typescript
} else if (isMemberCall(expression)) {
    const memberRef = expression.member.$refText;
    if (memberRef === 'class' || memberRef === 'CLASS') {
        // Java .class idiom — return java.lang.Class
        return this.javaInterop.getResolvedClass('java.lang.Class');
    }
    // ... existing logic
}
```

**Files touched:** `bbj-type-inferer.ts` (primary), `bbj-scope.ts` if `.class` needs to resolve in scope.

### H. Constructor Completion for `new ClassName()`

**Current behavior:** When a user types `new MyClass(`, the cursor is inside the `ConstructorCall.args` list. Langium's default completion provider does not know about constructor signatures — it falls back to generic scope completion.

**Fix:** Add a `ConstructorCall` detection in `BBjCompletionProvider.getCompletion()`. When the cursor is in a `ConstructorCall` context, look up the class's constructors (for Java classes: methods named the same as the simple class name; for BBj classes: `MethodDecl` nodes whose name matches the class name) and offer them as completion items with parameter snippet syntax.

**Files touched:** `bbj-completion-provider.ts` — add constructor completion interceptor.

### I. Deprecated Method Visual Indicator in Completion

**Current state (verified):**
- `MethodInfo.java` has no `deprecated` field
- `InteropService.java` does not inspect `@Deprecated` annotation
- `java-types.langium` `JavaMethod` interface has no `deprecated` property
- `BBjCompletionProvider` does not set `CompletionItemTag.Deprecated`

**Fix requires changes across the full stack:**

1. `MethodInfo.java`: add `public boolean deprecated;`
2. `InteropService.java` in `loadClassInfo()`: add `mi.deprecated = m.isAnnotationPresent(Deprecated.class);`
3. `java-types.langium`: add `deprecated?: boolean` to `JavaMethod` interface
4. `java-interop.ts` in `resolveClass()`: propagate `(method as Mutable<JavaMethod>).deprecated = method.deprecated`
5. `bbj-completion-provider.ts` in `createReferenceCompletionItem()`:
```typescript
if (nodeDescription.node && 'deprecated' in nodeDescription.node && nodeDescription.node.deprecated) {
    superImpl.tags = [CompletionItemTag.Deprecated];
}
```

**Files touched:** `MethodInfo.java`, `InteropService.java`, `java-types.langium`, `java-interop.ts`, `bbj-completion-provider.ts`.

---

## Recommended Project Structure (existing, unchanged)

```
bbj-vscode/src/language/
├── bbj.langium                   # Grammar (verbs, terminals, types)
├── java-types.langium            # Java interop type declarations
├── bbj-token-builder.ts          # Chevrotain token assembly + LONGER_ALT
├── bbj-lexer.ts                  # Line-continuation pre-processing
├── bbj-module.ts                 # DI wiring
├── bbj-scope.ts                  # Global + member scope resolution
├── bbj-scope-local.ts            # Local symbols computation
├── bbj-type-inferer.ts           # Expression -> Class inference
├── bbj-validator.ts              # Semantic validation
├── bbj-document-validator.ts     # Diagnostic hierarchy/suppression
├── bbj-completion-provider.ts    # LSP completion items
├── java-interop.ts               # Java backend service client
├── bbj-ws-manager.ts             # Workspace + settings lifecycle
└── validations/
    ├── check-classes.ts
    ├── check-variable-scoping.ts
    └── line-break-validation.ts
bbj-vscode/tools/
├── web.bbj                       # BUI/DWC launcher (fix "--" guard here)
├── em-login.bbj
└── em-validate-token.bbj
bbj-vscode/syntaxes/
├── bbj.tmLanguage.json           # TextMate grammar for BBj source
└── bbx.tmLanguage.json           # TextMate grammar for BBx config files (register in package.json)
java-interop/src/main/java/bbj/interop/
├── InteropService.java           # Add deprecated annotation inspection
└── data/
    └── MethodInfo.java           # Add deprecated field
```

---

## Architectural Patterns

### Pattern 1: Grammar-First Feature Addition (Verbs)

**What:** New BBj verbs are added by writing the grammar rule, adding it to `SingleStatement` alternatives, running `langium generate`, then adding a test.

**When to use:** EXIT int, SERIAL, ADDR.

**Trade-offs:** Grammar changes regenerate `generated/ast.ts` and `generated/grammar.ts`. The grammar is the single source of truth — no additional service wiring is needed unless the new construct requires validation or completion enhancements.

**Example (EXIT with optional int):**
```
ExitWithNumberStatement:
    (kind='EXIT' exitVal=Expression?)
    | RELEASE_NL
    | RELEASE_NO_NL exitVal=Expression
;
```

### Pattern 2: Token Builder LONGER_ALT Fix (Lexer Bugs)

**What:** BBj keywords that tokenize before identifiers cause problems when the keyword text appears as a prefix of a valid identifier. The fix ensures `LONGER_ALT = [idWithSuffix, id]` for all such tokens.

**When to use:** `#379` (`releaseVersion!` bug). Pattern established in v3.4 for `step` prefix fix.

**Example:**
```typescript
// Current (broken for suffixed identifiers):
releaseNl.LONGER_ALT = id;
// Fixed:
releaseNl.LONGER_ALT = [idWithSuffix, id];
```

### Pattern 3: Scope Provider Receiver-Type Dispatch (Completion Enhancements)

**What:** When resolving member access scope, `BbjScopeProvider.getScope()` dispatches on receiver type. Adding static-only completion means detecting when the receiver is a class reference (not an instance) and filtering to static members only.

**When to use:** `#374` (static method completion).

**Trade-offs:** The type inferer already returns the `Class` node for a bare `SymbolRef` referencing a class (via `isClass(reference)` branch). The receiver classification check is: `isSymbolRef(receiver) && isClass(receiver.symbol.ref)`.

### Pattern 4: Completion Provider Javadoc + Tag Enrichment

**What:** `BBjCompletionProvider.createReferenceCompletionItem()` reads node metadata set by `java-interop.ts` during `resolveClass()`. Adding deprecated indicators follows the same read-from-node pattern used for `docu` (Javadoc).

**When to use:** Deprecated method indicator.

**Trade-offs:** Deprecated flag must flow through java-interop boundary: `MethodInfo.java` -> `JavaMethod` interface -> `resolveClass()` -> completion item. Requires a Java build.

### Pattern 5: TextMate Grammar Registration (Highlighting Fix)

**What:** VS Code TextMate grammars are registered in `package.json` under `"grammars"`. A grammar file with a distinct scopeName can be added alongside the existing BBj grammar.

**When to use:** `#381` (config.bbx highlighting).

**Trade-offs:** Registering `bbx.tmLanguage.json` for the `bbj` language id may affect ALL `.bbx` files, not just `config.bbx`. If `.bbx` files are BBj source programs (they are), they should use `bbj.tmLanguage.json`, not `bbx.tmLanguage.json`. The approach needs careful scoping — possibly a filename-based association rather than extension-based.

---

## Data Flow

### Grammar Verb Addition Flow

```
bbj.langium (new rule added to grammar)
    |
    v (langium generate)
generated/ast.ts + generated/grammar.ts
    |
    v (TypeScript compilation + Chevrotain)
Parser recognizes new construct
    |
    v (parse)
New AST node type available in validators and services
```

### Completion Enhancement Flow (Static Methods)

```
User types "MyClass."
    |
    v
BBjCompletionProvider.getCompletion()
    |
    v (delegates to DefaultCompletionProvider)
BbjScopeProvider.getScope() { context.property === 'member' }
    |
    v
typeInferer.getType(receiver) -> BbjClass (isClass(reference))
    |
    v (detect: isSymbolRef(receiver) && isClass(receiver.symbol.ref))
createBBjClassStaticMemberScope(bbjType)
    |
    v (filter: member.static === true)
Completion items showing only static methods/fields
```

### Deprecated Method Flow

```
InteropService.java: m.isAnnotationPresent(Deprecated.class) -> mi.deprecated = true
    |
    v JSON-RPC response
java-interop.ts: resolveClass()
    |
    v (method as Mutable<JavaMethod>).deprecated = rawMethod.deprecated
java-types.langium: JavaMethod.deprecated?: boolean
    |
    v
bbj-completion-provider.ts: createReferenceCompletionItem()
    |
    v node.deprecated -> tags: [CompletionItemTag.Deprecated]
IDE completion list (strikethrough on deprecated items)
```

### EM Config Fix Flow (web.bbj)

```
VS Code: Commands.cjs runWeb()
    -> configPath from settings (may be "" or a path)
    -> cmd args: ... "${token}" "${configPath}"

web.bbj ARGV(9) = configFile!
    -> CURRENT: passes any value including "--" to app!.setString(CONFIG_FILE)
    -> FIXED: guard against "--" same as sscp is guarded
    -> if configFile == "--" or empty: use BBjAPI().getConfig().getConfigFileName()
```

---

## Suggested Build Order

Dependencies flow from most-isolated to most-coupled:

**Phase 1 — Isolated fixes (single-file, no grammar regen needed):**
1. `releaseVersion!` (#379) — `bbj-token-builder.ts` only, 2-line LONGER_ALT fix. Highest confidence, lowest risk, zero grammar regeneration.
2. config.bbx highlighting (#381) — `package.json` only, add grammar entry. No code change.
3. EM Config "--" fix (#382) — `tools/web.bbj` guard. Self-contained.

**Phase 2 — Grammar-only additions (require `langium generate`):**
4. EXIT int verb (#376) — grammar rule update, test.
5. SERIAL verb (#375) — new grammar rule + SingleStatement entry, test.
6. ADDR fix (#377) — verify/update AddrStatement rule, test.
7. DECLARE in class (#380) — grammar ClassDecl alternative.

**Phase 3 — Completion enhancements (depend on existing type inferer, no java-interop change):**
8. Static methods (#374) — scope provider + optional completion provider update.
9. `.class` property (#373) — type inferer branch addition.
10. Constructor completion — completion provider interceptor.

**Phase 4 — Cross-layer change (requires Java build + Langium interface + LS change):**
11. Deprecated method indicator — Java reflection change, grammar interface, resolver, completion. Highest coupling.

---

## Integration Points: New vs. Modified

No new files are required for any v3.9 feature. All changes integrate into existing files.

| Feature | Files Modified | Change Scope |
|---------|---------------|--------------|
| EXIT int verb (#376) | `bbj.langium` | Grammar rule update |
| SERIAL verb (#375) | `bbj.langium` | New grammar rule + SingleStatement |
| ADDR fix (#377) | `bbj.langium` | Verify/fix AddrStatement rule |
| `releaseVersion!` fix (#379) | `bbj-token-builder.ts` | 2 lines |
| DECLARE in class (#380) | `bbj.langium` | ClassDecl grammar alternative |
| EM Config "--" fix (#382) | `tools/web.bbj` | Guard configFile against "--" |
| config.bbx highlighting (#381) | `bbj-vscode/package.json` | Add grammar entry for bbx.tmLanguage.json |
| Static methods (#374) | `bbj-scope.ts` | Scope filtering for class receivers |
| `.class` property (#373) | `bbj-type-inferer.ts` | Special-case branch for `.class` |
| Constructor completion | `bbj-completion-provider.ts` | New completion interceptor |
| Deprecated indicator | `MethodInfo.java`, `InteropService.java`, `java-types.langium`, `java-interop.ts`, `bbj-completion-provider.ts` | Full cross-layer change |

---

## Anti-Patterns

### Anti-Pattern 1: Modifying Generated Files

**What people do:** Edit `generated/ast.ts` or `generated/grammar.ts` directly.
**Why it's wrong:** These files are overwritten on every `langium generate` run.
**Do this instead:** Modify `bbj.langium` or `java-types.langium` and re-run `langium generate`.

### Anti-Pattern 2: Adding LONGER_ALT Without idWithSuffix

**What people do:** When fixing a keyword-as-identifier tokenization bug, set `LONGER_ALT = id` only.
**Why it's wrong:** Identifiers with `!`, `$`, or `%` suffixes still break because `idWithSuffix` is not checked first.
**Do this instead:** Always set `LONGER_ALT = [idWithSuffix, id]` for all keyword tokens that can appear as identifier prefixes. This is the bug in #379.

### Anti-Pattern 3: Scope Resolution Without Cycle Protection

**What people do:** Traverse `bbjType.extends` chain recursively without depth/visited guards.
**Why it's wrong:** Cyclic inheritance (or parser-recovered partial ASTs) causes infinite loops.
**Do this instead:** Always pass and check a `visited: Set<BbjClass>` and `depth < MAX_INHERITANCE_DEPTH` guard, as established throughout `bbj-scope.ts` and `bbj-completion-provider.ts`.

### Anti-Pattern 4: Completion Items Without Snippet Syntax on Methods

**What people do:** Return a plain label for method completion without `insertText` using snippet syntax.
**Why it's wrong:** Users must manually type parameter placeholders; tab-stop navigation is unavailable.
**Do this instead:** Follow the pattern in `createReferenceCompletionItem()`: set `insertTextFormat = 2` and `insertText = label((p, i) => "${" + (i + 1) + ":" + p + "}")`.

### Anti-Pattern 5: Direct Import of main.ts in Service Files

**What people do:** Import from `main.ts` to access the LSP connection for notifications.
**Why it's wrong:** `createConnection` executes at module load time; importing `main.ts` from service files crashes Vitest workers.
**Do this instead:** Use the `bbj-notifications.ts` isolation module for all notification sending. Never import from `main.ts` in language service files.

---

## Scaling Considerations

This is developer tooling (one LS process per IDE window). Concerns are per-document performance, not horizontal scale.

| Concern | Current state | Watch for in v3.9 |
|---------|--------------|-----------|
| CPU in multi-project workspaces | Known issue #232, mitigations documented | Static member scope filtering adds traversal; keep depth-bounded |
| Java class resolution | Mutex + dedup + 20-depth + 30s timeout | Deprecated field adds one boolean; negligible overhead |
| Grammar complexity | 47 known Chevrotain ambiguities, all safe | Each new verb may add ambiguity; test with parser |
| Completion latency | Javadoc in completion items added in v3.8 | Constructor completion adds class lookup; monitor in tests |

---

## Sources

All findings from direct codebase inspection (HIGH confidence):

- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj.langium` — grammar, ExitWithNumberStatement, AddrStatement
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj-token-builder.ts` — LONGER_ALT patterns, RELEASE_NL/RELEASE_NO_NL gap
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj-completion-provider.ts` — completion logic, Javadoc integration
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj-scope.ts` — member scope dispatch, createBBjClassMemberScope
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj-type-inferer.ts` — type inference, isClass branch
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/java-interop.ts` — resolveClass, DocumentationInfo population
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/java-types.langium` — JavaMethod interface (no deprecated field)
- `/Users/beff/_workspace/bbj-language-server/java-interop/src/main/java/bbj/interop/data/MethodInfo.java` — confirmed: no deprecated field
- `/Users/beff/_workspace/bbj-language-server/java-interop/src/main/java/bbj/interop/InteropService.java` — confirmed: no @Deprecated inspection
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/tools/web.bbj` — "--" guard for sscp, missing guard for configFile
- `/Users/beff/_workspace/bbj-language-server/bbj-intellij/src/main/java/com/basis/bbj/intellij/actions/BbjRunActionBase.java` — getConfigPath(), getConfigPathArg()
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/package.json` — grammar registrations (bbx.tmLanguage.json not registered)
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/syntaxes/bbx.tmLanguage.json` — exists, scopeName: "source.bbx", not wired up

---

*Architecture research for: BBj Language Server v3.9 Quick Wins*
*Researched: 2026-02-20*
