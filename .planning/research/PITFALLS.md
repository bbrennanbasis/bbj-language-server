# Pitfalls Research

**Domain:** Adding grammar verbs, fixing tokenizer bugs, and adding completion enhancements to an existing Langium/Chevrotain language server (BBj LS v3.9 milestone)
**Researched:** 2026-02-20
**Confidence:** HIGH (codebase analysis of 300+ grammar rules, token-builder, completion provider, scope provider, and 16 prior milestone decisions)

---

## Critical Pitfalls

### Pitfall 1: New Keyword Breaks Existing Identifiers That Start With That Word

**What goes wrong:**
Adding a new keyword (e.g., `SERIAL`, `ADDR` with a new signature, `EXIT` with an int) causes identifiers that share the same prefix to tokenize as the new keyword instead of as `ID`. For example, adding `SERIAL` as a keyword will cause the variable `serialNumber!` to tokenize as `SERIAL` + `Number!` (two tokens), producing a parser error on any file that uses such a variable. This is the exact class of bug that was fixed for `stepXYZ!` in v3.4 and `mode$` in v3.2.

**Why it happens:**
Chevrotain's tokenizer is greedy and keyword-biased. Without `LONGER_ALT`, a keyword like `SERIAL` matches before the scanner can check that the next character continues the identifier. The `BBjTokenBuilder` applies `LONGER_ALT = [idWithSuffix, id]` to all uppercased keywords automatically (line 48 of `bbj-token-builder.ts`), but this only prevents suffix-bearing collisions (`serial!`). A keyword prefix match against a plain `ID` without suffix (e.g., a variable named `serial`) still needs the `[idWithSuffix, id]` ordering to fire correctly. If the `CATEGORIES` or `LONGER_ALT` arrays are modified incorrectly for the new keyword, the protection breaks.

**How to avoid:**
1. Verify that the new keyword is matched by the regex `[A-Z]+(?!_)` in `BBjTokenBuilder.buildTokens()` so it receives the automatic `LONGER_ALT = [idWithSuffix, id]` assignment. Do not add it to `EXCLUDED`.
2. Write a parser test using the keyword text as a variable name with each suffix type: `serial!`, `serial$`, `serial%`, and plain `serial` as a field name, method name, and standalone variable.
3. Run the full `example-files.test.ts` suite after the grammar change — it exercises real-world BBj programs that commonly use keyword-like identifiers.

**Warning signs:**
- Parser test for the new verb works, but `example-files.test.ts` starts failing.
- Identifiers like `serialPort!`, `serialNumber$`, or any identifier beginning with the new keyword text produce unexpected lexer errors.
- Chevrotain emits an ambiguity warning for the new keyword against `ID` in debug mode.

**Phase to address:** Grammar verb addition phase (first phase of v3.9).

---

### Pitfall 2: `releaseVersion!` Suffix Tokenization — LONGER_ALT Array Order Matters

**What goes wrong:**
The bug `#379 releaseVersion! flagged as parse error` arises because the `RELEASE_NL` and `RELEASE_NO_NL` terminal tokens use `LONGER_ALT = id` (singular, line 56-57 of `bbj-token-builder.ts`) instead of `LONGER_ALT = [idWithSuffix, id]`. When the identifier `releaseVersion!` is encountered, the tokenizer matches `RELEASE` against `RELEASE_NO_NL`, then the `idWithSuffix` check is not tried because `LONGER_ALT` is only `id` (not the array). The `!` suffix is orphaned and causes a parse error. This is the root cause pattern.

**Why it happens:**
`RELEASE_NL` and `RELEASE_NO_NL` are custom terminal tokens built in `buildTerminalToken()` with regex patterns (`/RELEASE(?=\s*(;\s*|\r?\n))/i` and `/RELEASE(?!\s*(;\s*|\r?\n))/i`). They are patched with `LONGER_ALT = id` (singular) rather than `[idWithSuffix, id]` (array). The v3.4 fix for generic keywords updated the keyword loop but did not update these two special cases. When `releaseVersion!` appears in code that is NOT followed by a newline or semicolon, `RELEASE_NO_NL` fires but its singular `LONGER_ALT` only checks `id` (no suffix), leaving `!` unmatched.

**How to avoid:**
Change both RELEASE special cases in `bbj-token-builder.ts` from:
```ts
releaseNl.LONGER_ALT = id;
releaseNoNl.LONGER_ALT = id;
```
to:
```ts
releaseNl.LONGER_ALT = [idWithSuffix, id];
releaseNoNl.LONGER_ALT = [idWithSuffix, id];
```
Write a dedicated test: variable named `releaseVersion!`, `releaseDate$`, `releaseCount%`, and a method named `release()` — all in the same file to exercise both `RELEASE_NL` (standalone) and `RELEASE_NO_NL` (inline) code paths.

**Warning signs:**
- Any variable that begins with `release` and has a `!`, `$`, or `%` suffix causes a parser/lexer error.
- The error message mentions an unexpected token at the `!` position.
- `RELEASE` as a standalone statement works, but `releaseVersion! = "1.0"` fails.

**Phase to address:** Parser bug fix phase (fix #379).

---

### Pitfall 3: `DECLARE` Outside a Method Body Reaches a Grammar Rule That Expects It Only Inside `ClassMember`

**What goes wrong:**
Bug `#380 DECLARE in class outside methods breaks parser`. The grammar defines `ClassMember: FieldDecl | MethodDecl`. `VariableDecl` (which uses the `declare` keyword) is a `SingleStatement` reachable only through `Statement → SingleStatement → VariableDecl`. Inside a class body, the parser expects only `ClassMember | Comments`, so a bare `declare` keyword at class scope triggers a parser error cascade. The cascade then reports errors on subsequent `FIELD` and `METHOD` declarations inside the same class, producing 10-40 noise diagnostics.

**Why it happens:**
Real-world BBj class files sometimes have `DECLARE` statements at the class body level (outside any method), possibly as a coding style artifact or editor paste. The grammar strictly requires `declare` inside method bodies, but the diagnostic suppression system introduced in v3.7 cannot suppress all cascade errors because they span across class member boundaries.

**How to avoid:**
Two safe approaches (choose one per the milestone design):
1. **Tolerate-and-ignore approach**: Allow `VariableDecl` inside the `ClassMember` union in the grammar (add it as an optional alternative). This is the simplest grammar change. The semantic validator can then produce a targeted warning: "DECLARE is not allowed at class scope — move inside a method."
2. **Error recovery approach**: Leave the grammar strict. Ensure `suppressCascading` logic in `bbj-document-validator.ts` correctly filters the cascade when a class-scope parse error is the root. The per-file suppression introduced in v3.7 applies, so a single root cause error should suppress follow-on class member errors.

Do NOT mix both approaches — either recover in grammar or recover in the diagnostic layer. Mixing creates double-reporting of the same error.

**Warning signs:**
- A parse test for `DECLARE` inside a class shows one error in test output but users report 30+ errors in the IDE.
- The cascade suppression test in `validation.test.ts` passes but does not cover the class-scope case.
- `VariableDecl` appears in class-level error traces but has no specific `ClassMember` grammar production.

**Phase to address:** Parser bug fix phase (fix #380).

---

### Pitfall 4: Generated AST Must Be Regenerated After Grammar Changes

**What goes wrong:**
After modifying `bbj.langium` (adding `SERIAL`, `EXIT int?`, changing `AddrStatement`, adding grammar alternatives), the generated files in `bbj-vscode/src/language/generated/` (`ast.ts`, `grammar.ts`) become stale. TypeScript compilation succeeds because the old generated types still match the old grammar structure, but at runtime Langium uses the stale grammar metadata. The result is that new grammar rules are silently ignored or cause internal Langium assertion errors during parsing.

**Why it happens:**
Langium's generator (`langium-cli generate`) must be run explicitly after grammar changes. The build pipeline (esbuild/npm) does not watch `*.langium` files for changes. The generated files look like source code and are committed to the repo, so a developer editing the grammar without running the generator will produce committed code that passes CI (TypeScript compiles) but fails at parse time.

**How to avoid:**
Always run `npm run langium:generate` (or the equivalent `langium generate` CLI command) immediately after editing `bbj.langium`. Then verify the generated `ast.ts` contains the new types (e.g., `isSerialStatement`, updated `ExitWithNumberStatement` interface with optional `exitVal`). Run `npm run build` and the test suite before committing.

Add a CI check: `git diff --exit-code src/language/generated/` after running the generator — if generated files differ from what is committed, fail the build.

**Warning signs:**
- TypeScript compiles with zero errors after grammar change, but parser tests fail with "unexpected token" on the new construct.
- The new grammar rule node type is missing from `generated/ast.ts` after a grammar edit.
- Langium emits a runtime warning about unrecognized rule names.
- `expectToContainAstNodeType(result, isSerialStatement)` test helper throws "isSerialStatement is not a function".

**Phase to address:** Any grammar-change phase — must be the first action after any `bbj.langium` edit.

---

### Pitfall 5: Static Method Completion Conflates Instance Scope with Class Scope

**What goes wrong:**
When implementing static method completion for `USE`-imported classes (feature #374), there is a risk of returning both static and instance members when the receiver resolves to a class reference (not an instance). The existing `createBBjClassMemberScope()` in `bbj-scope.ts` and the `MemberCall` branch in `getScope()` both assume the receiver is an instance. If the type inferer returns a `BbjClass` node when `ClassName` (a USE reference) is used as a receiver, the scope provider returns all members — static and instance — mixed together. Users see instance methods on class references, which is misleading and can cause false-positive completion suggestions.

**Why it happens:**
`BBjTypeInferer.getType()` handles `isClass(reference)` by returning `reference` directly (line 43 of `bbj-type-inferer.ts`). When a user writes `MyClass.someStaticMethod()`, the `SymbolRef` for `MyClass` resolves to the class node itself, and `getType()` returns the class. The `MemberCall` branch in `getScope()` then hits `isBbjClass(receiverType)` and calls `createBBjClassMemberScope(receiverType)`, which returns all members. The fix requires distinguishing "receiver is the class itself" (static context) from "receiver is an instance of the class" (instance context).

**How to avoid:**
Introduce a discriminator: when the receiver expression is a `SymbolRef` whose resolved `reference` is a `BbjClass` (not a `VariableDecl`, `FieldDecl`, or `Assignment`), treat it as a class reference and return only `static` members. The `FieldDecl` and `MethodDecl` nodes already carry a `static` boolean from the grammar fragment `Static: static?='STATIC'?`. Filter using `member.static === true`.

Do not change the type inferer — its job is to return the type. The scope provider, which already has different branches for different reference types, is the correct place to add static vs. instance filtering.

**Warning signs:**
- Completion after `MyClass.` shows all methods including non-static ones.
- A test for static method completion also returns instance methods in the completion list.
- Instance methods appear with different icons than expected in the completion popup.

**Phase to address:** Static method completion phase (feature #374).

---

### Pitfall 6: `.class` Property Conflicts With Langium's Cross-Reference Mechanism

**What goes wrong:**
Implementing `.class` property resolution (feature #373) is tempting to implement as a special string comparison in the `MemberCall` scope provider branch: "if member name is `class`, return the java.lang.Class type." However, `.class` in the grammar's `MemberCall` rule resolves `member` as a cross-reference to a `NamedElement`. Since `class` is also a keyword in `bbj.langium` (the `CLASS` statement), the tokenizer tokenizes `.class` as `.` + `CLASS_KEYWORD`, not as `.` + `ID`. This means the cross-reference lookup for member `class` never fires — the grammar rejects it before the scope provider is reached.

**Why it happens:**
`CLASS` appears in the grammar as a keyword for `ClassDecl` and `CLASSEND` is excluded from `LONGER_ALT`. In the `MemberCall` rule, `member=[NamedElement:FeatureName]` expects a `FeatureName` (which is `ID | ID_WITH_SUFFIX`). Since `CLASS` tokenizes as a keyword, not as `ID`, the reference parser fails immediately. The grammar needs to explicitly allow `class` (case-insensitive) in `FeatureName` or as a special alternative in the `MemberCall` rule.

**How to avoid:**
Add `class` as an alternative in `FeatureName` (or in `ValidName`) explicitly: `FeatureName returns string: ID | ID_WITH_SUFFIX | 'CLASS'`. Alternatively, handle `.class` as a special `PrimaryExpression`/`MemberCall` production that bypasses normal member lookup. The scope provider then needs a special case: when `member.$refText.toUpperCase() === 'CLASS'`, return `java.lang.Class` from the java-interop resolved classes.

Do not implement this as a pure runtime hack in the scope provider without the grammar fix — the grammar will reject it before the scope provider is ever called.

**Warning signs:**
- Parser error on `myObj!.class` before any scope provider code executes.
- Test for `.class` completion shows "unexpected token 'CLASS'" in parser errors array.
- The `member` cross-reference in MemberCall has `$refText === ''` or never resolves.

**Phase to address:** `.class` property resolution phase (feature #373).

---

### Pitfall 7: Constructor Completion Fires Too Eagerly Outside `new` Context

**What goes wrong:**
When adding constructor completion for `new ClassName(args)` (feature), the completion provider may fire for all class name references, not just those following the `new` keyword. This results in constructor snippets appearing in `USE` statements, `CAST` expressions, type annotations in `DECLARE`, `FIELD`, and method return types — all places where a class name is typed but a constructor call is not appropriate.

**Why it happens:**
The `DefaultCompletionProvider.completionFor()` mechanism in Langium triggers on any grammar position that accepts a `QualifiedClass` or `Class` reference. The `ConstructorCall` grammar rule has `klass=QualifiedClass`, making it one of many places that can produce a class reference completion. Without a context guard that checks whether the immediate grammar parent is `ConstructorCall`, the constructor snippet appears everywhere a class can be typed.

**How to avoid:**
In the completion provider override, add a context check: inspect `context.node` and its `$container` chain. Only inject constructor snippets when the grammar path includes a `ConstructorCall` node or the cursor is immediately preceded by the `new` keyword. Use `AstUtils.getContainerOfType(context.node, isConstructorCall)` as the guard. Alternatively, implement constructor completion as a trigger on the space following `new` — filter for contexts where the keyword `new` is the last token before the cursor.

**Warning signs:**
- Constructor snippet appears in the `USE ::path::` class reference completion list.
- Constructor snippet appears after `CAST(` which is a class reference context but not a constructor context.
- Test for USE completion shows constructor items mixed with class name items.

**Phase to address:** Constructor completion phase.

---

### Pitfall 8: Deprecated Method Tag Uses Wrong LSP Field and Is Stripped by Langium

**What goes wrong:**
LSP `CompletionItem.tags` with `CompletionItemTag.Deprecated = 1` is the correct way to show strikethrough on deprecated methods in VS Code and IntelliJ. However, Langium's `DefaultCompletionProvider.createReferenceCompletionItem()` creates a `CompletionValueItem` (Langium's internal type) that maps to `CompletionItem` only after `fillCompletionItem()` runs. If `tags` is not explicitly set on the `CompletionValueItem` before `fillCompletionItem()`, the field is lost. Additionally, the existing `bbj-completion-provider.ts` overrides `createReferenceCompletionItem()` but only sets `kind`, `sortText`, `label`, `labelDetails`, `insertText`, and `documentation` — `tags` is not carried over.

**Why it happens:**
`CompletionValueItem` in Langium 4.x maps to LSP `CompletionItem` but does not include `tags` in its default field mapping. The `BBjCompletionProvider.createReferenceCompletionItem()` calls `super.createReferenceCompletionItem()` and then mutates the result — but `tags` is only passed through if the parent impl sets it, which it does not for custom node types. The deprecated flag must be applied after `super()` is called, by checking `nodeDescription.node` for a `deprecated` marker.

**How to avoid:**
After the `super.createReferenceCompletionItem()` call, check if the node has a deprecated indicator (from Javadoc `@deprecated` tag stored during class resolution in `java-interop.ts`). If so, set `superImpl.tags = [CompletionItemTag.Deprecated]`. Ensure the `DocumentationInfo` interface (in `generated/ast.ts` types or as an augmentation in `java-interop.ts`) carries a `deprecated: boolean` field populated during the Javadoc parsing step in `resolveAndLinkClass()`.

Import `CompletionItemTag` from `vscode-languageserver`, not from `vscode` — the language server runs in Node.js without VS Code extension APIs.

**Warning signs:**
- Methods with `@deprecated` Javadoc show in completions without strikethrough in VS Code.
- `CompletionItemTag is not defined` runtime error if imported from wrong module.
- Strikethrough works in VS Code but not IntelliJ — check that LSP4IJ passes `tags` through (it should, but verify with a debug completion response inspection).

**Phase to address:** Deprecated method indicator phase.

---

### Pitfall 9: EM Config `"--"` Argument Interpreted as Flag, Not Separator

**What goes wrong:**
Bug `#382 EM Config "--" causing DWC/BUI startup failure`. The `web.bbj` runner receives a `--` argument as a command-line separator (BBj's convention for separating runner options from program arguments). When the EM Config field in settings is set to `"--"` (two dashes), the value is passed as an argument to the `bbj` process. The `bbj` executable interprets `--` as "end of BBj options, start of program arguments" and either ignores subsequent arguments or enters an unexpected parsing mode, causing the BUI/DWC launch to fail silently.

**Why it happens:**
The `BbjRunBuiAction` and `BbjRunDwcAction` build a `GeneralCommandLine` by calling `cmd.addParameter()` for each argument sequentially, including the configPath. If `configPath` contains `--` (or the emUrl field contains it), `GeneralCommandLine.addParameter()` on IntelliJ passes it verbatim to the process. The BBj interpreter sees `--` as its argument separator before encountering the program path argument, discarding everything after it.

**How to avoid:**
Validate the configPath and emUrl settings fields on save in `BbjSettingsConfigurable.apply()`. If the value equals `--` or starts with `-` unexpectedly, show a validation error before applying. Additionally, sanitize in the action builder: if `configPath.trim() === '--'` or `emUrl.trim() === '--'`, treat as empty/default and log a warning. Do not pass the raw setting value to `addParameter()` without validation.

**Warning signs:**
- DWC/BUI run produces no output in the log window and exits immediately with code 0 or 1.
- Removing the `--` from the EM Config setting makes run succeed.
- The `bbj` process exits before the web.bbj runner loads, visible only in debug-level LS logs.

**Phase to address:** Bug fix phase for #382.

---

### Pitfall 10: `config.bbx` File Loses Syntax Highlighting After Extension Update

**What goes wrong:**
Bug `#381 config.bbx file no longer highlighted`. The `config.bbx` file is associated with the `bbx` language ID in the TextMate grammar (`bbx.tmLanguage.json`, scopeName `source.bbx`), but the `package.json` grammar contribution only registers `source.bbj` for the `bbj` language ID. The `.bbx` extension is registered under the `bbj` language in `package.json` (line 34) — so `.bbx` files are typed as `bbj`, not as `bbx`. This means `config.bbx` is parsed by the BBj Langium grammar (expecting BBj program syntax) instead of the BBx property-file grammar.

**Why it happens:**
The `config.bbx` file is a BBx configuration file (key=value format), not a BBj program. It has its own TextMate grammar (`bbx.tmLanguage.json`) that covers `PREFIX`, `ALIAS`, and other config directives. But when `.bbx` is listed under the `bbj` language extensions (to support `.bbx` as BBj program files for run commands), the `config.bbx` file is accidentally reclassified as a BBj program. The BBj Langium parser then attempts to parse it and produces false errors.

**How to avoid:**
There are two valid approaches:
1. **File-name-specific association**: Register `config.bbx` by exact filename in `package.json` under a `bbx` language entry (separate from `bbj`), while keeping the `*.bbx` glob under `bbj`. VS Code supports `"files"` with exact filename in language configuration.
2. **Embedding grammar approach**: Keep `.bbx` under `bbj` but use VS Code's `"embeddedLanguages"` or override the TextMate grammar for files whose name matches `config.bbx` using a `languages[].filenamePatterns` entry.

Do not remove `.bbx` from the `bbj` language entirely — that would break run commands for `.bbx` programs. The fix must be additive (config.bbx gets bbx grammar, other .bbx files stay as bbj).

**Warning signs:**
- Opening `config.bbx` shows red squiggles from the BBj parser on `PREFIX` and `ALIAS` lines.
- Syntax highlighting in `config.bbx` shows BBj keyword coloring instead of the bbx property-file coloring.
- The `bbx.tmLanguage.json` file exists in syntaxes/ but has no grammar contribution entry in `package.json`.

**Phase to address:** Bug fix phase for #381.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Add keyword to grammar without adding to KEYWORD_STANDALONE regex | New verb works for most cases | Verb followed by newline with no args silently fails to tokenize correctly | Never — always check if verb can appear standalone |
| Hardcode static method filter in completion without updating type inferer | Faster to implement | Type inferer used by 4+ consumers returns wrong type for static calls, causing future inference bugs | Never — fix in scope provider only, not type inferer |
| Skip regenerating `generated/ast.ts` after minor grammar tweak | Faster iteration loop | Runtime-only failures that bypass TypeScript type checking | Never — always regenerate |
| Copy-paste completion item construction for deprecated markers | Avoids refactor | Duplicated logic diverges when `DocumentationInfo` structure changes | Acceptable only if isolated to a single method |
| Use `string.includes('deprecated')` on Javadoc text to detect deprecated | No AST change needed | Brittle — matches `"not deprecated"`, comment text, parameter names | Never — parse the `@deprecated` JSDoc tag |

---

## Integration Gotchas

Common mistakes when connecting features to existing services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `java-interop.ts` deprecated detection | Check `method.deprecated` field that doesn't exist on `JavaMethod` type | Add `deprecated: boolean` to `DocumentationInfo` (already has `javadoc` and `signature`) and populate it during Javadoc parsing in `resolveAndLinkClass()` |
| Static method scope via `BbjScopeProvider` | Add new branch in `getScope()` that duplicates `createBBjClassMemberScope()` call | Reuse `createBBjClassMemberScope()` with a `staticOnly: boolean` parameter to filter `member.static` |
| `bbj-token-builder.ts` RELEASE terminals | Treating them identically to regular keyword tokens in the `LONGER_ALT` patch loop | These are manually patched outside the keyword loop (lines 52-57); must be updated separately |
| Grammar change and TextMate grammar | Updating Langium grammar but forgetting TextMate grammar | New keywords added to `.langium` must also be added to `bbj.tmLanguage.json` for syntax highlighting to work in the IDE (the two are independent) |
| `config.bbx` file type fix | Registering `bbx` grammar contribution in IntelliJ's `textmateBundles` XML only | Must also update VS Code `package.json` grammars contribution; both IDEs share the TextMate file but register it independently |
| Constructor completion filtering | Using `context.node.$type === 'ConstructorCall'` | At completion time the AST node may be partially constructed; use CST parent traversal or keyword lookahead instead |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Static method completion iterates all members of all USE-imported classes | Completion popup delays 200-500ms on files with 10+ USE statements | Cache the `static` members per class in the `JavaInteropService` or `BbjScopeProvider`; filter at construction not at response time | Files with 20+ USE statements and large Java classpath |
| `.class` property lookup triggers full `getResolvedClass('java.lang.Class')` on every member access | Slow completion and hover on files with many member calls | Resolve `java.lang.Class` once at startup and cache the reference; reuse the cached reference | Any file with 5+ member access chains |
| Deprecated marker detection parses raw Javadoc text on every completion request | Completion latency proportional to Javadoc text length | Populate `deprecated` flag once during `resolveAndLinkClass()` and store on `DocumentationInfo` | Classes with long Javadoc (BBjAPI has extensive docs) |

---

## UX Pitfalls

Common user experience mistakes in this domain.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Constructor snippet for abstract classes | Users select completion, get snippet, then see "cannot instantiate abstract class" error | Suppress constructor completion for Java classes flagged as abstract (`javaClass.abstract` if available from java-interop) |
| Deprecated strikethrough without tooltip explaining why | Users see strikethrough but don't know if they should migrate | Include the `@deprecated` Javadoc text in the `documentation` field of the completion item, not just the `tags` field |
| Static method completion on instances (not just class references) | Users discover static methods only when typing on a class reference, not on an instance | Consider whether to also show static methods on instance receiver completions (Java allows this at runtime; BBj may too) — document the decision |
| `EXIT` with int parameter — no documentation in hover | Users don't know what the int argument means | Add `EXIT int` to the library BBj docs file or add hover documentation that explains the exit code semantics |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **SERIAL grammar rule**: After adding the verb, verify it appears in the TextMate grammar (`bbj.tmLanguage.json`) keyword list — otherwise no syntax highlighting in editor even if Langium parses correctly.
- [ ] **EXIT int parameter**: The grammar change makes `EXIT 0` parse, but `EXIT` standalone (no int) must still work via the existing `KEYWORD_STANDALONE` or the `kind='EXIT'` alternative. Verify both `EXIT` and `EXIT 1` in parser tests.
- [ ] **Static method completion**: After adding scope filtering, verify that auto-complete triggered on a USE class reference (`MyClass.|`) shows only static methods. Also verify that `new MyClass().|` (instance) shows only instance methods.
- [ ] **Deprecated indicator in IntelliJ**: VS Code uses `CompletionItemTag.Deprecated` which renders as strikethrough. Verify IntelliJ/LSP4IJ also renders strikethrough — it should since LSP4IJ implements the LSP spec, but test on a real IntelliJ instance.
- [ ] **`config.bbx` fix in IntelliJ**: The fix in VS Code `package.json` does not automatically apply to IntelliJ TextMate bundle registration. Check if IntelliJ's `textmateBundles` entry in plugin.xml needs a separate `fileNamePatterns` addition.
- [ ] **`releaseVersion!` fix regression**: After fixing `RELEASE_NO_NL` and `RELEASE_NL` LONGER_ALT, run the full set of RELEASE tests (lines 1968-2015 of `parser.test.ts`) to ensure standalone `RELEASE`, `RELEASE expr`, and `release` as field/method names all still work.
- [ ] **Constructor completion does not appear in USE statement**: After adding constructor snippet, open a new `.bbj` file, type `use ::path::` — verify no constructor snippets appear in the class reference completion list.

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Keyword breaks existing identifiers | MEDIUM | Revert grammar change, add targeted `LONGER_ALT` fix in `bbj-token-builder.ts`, re-run tests |
| Generated AST stale | LOW | Run `npm run langium:generate`, rebuild, re-run tests |
| Static + instance methods mixed in completion | LOW | Add `static` filter in scope provider branch; no grammar change needed |
| Deprecated tag import from wrong module | LOW | Change import from `vscode` to `vscode-languageserver`; rebuild |
| `.class` grammar rejection | MEDIUM | Add `'CLASS'` to `FeatureName` alternatives in grammar, regenerate AST, add test |
| EM Config `--` validation missing | LOW | Add sanitize step in action builder; no grammar change needed |
| `config.bbx` highlighting broken | MEDIUM | Add separate language registration in `package.json` for `config.bbx` by filename; update IntelliJ plugin.xml if needed |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| New keyword breaks identifiers | Grammar verb addition (SERIAL, ADDR, EXIT) | Parser test: keyword text as variable name with each suffix type |
| RELEASE suffix LONGER_ALT (releaseVersion!) | Bug fix #379 | Parser test with releaseVersion!, releaseDate$, releaseCount% |
| DECLARE at class scope (#380) | Bug fix #380 | Parser test: DECLARE inside class body, outside method |
| Stale generated AST | Every grammar-change phase (run langium:generate first) | Check `generated/ast.ts` for new node type |
| Static/instance conflation | Static method completion (#374) | Completion test: only static methods on class references |
| .class grammar rejection | .class property (#373) | Parser test: `myObj!.class` parses; scope test: resolves to java.lang.Class |
| Constructor completion scope leak | Constructor completion | Completion test: no constructor snippets in USE context |
| Deprecated tag wrong module | Deprecated indicator | TypeScript compiles without `vscode` dependency; strikethrough verified in VS Code and IntelliJ |
| EM Config `--` argument | Bug fix #382 | Settings validation test: `--` value shows error in UI |
| config.bbx highlighting (#381) | Bug fix #381 | Open config.bbx in VS Code: bbx grammar colors apply, no parser errors |

---

## Sources

- Codebase analysis: `bbj-token-builder.ts` (LONGER_ALT patterns, lines 39-57), `bbj-completion-provider.ts` (completion override), `bbj-scope.ts` (MemberCall branch, lines 154-165), `bbj.langium` (grammar rules for ExitWithNumberStatement, AddrStatement, FeatureName, ValidName)
- Prior milestone decisions from `PROJECT.md`: LONGER_ALT array [idWithSuffix, id] decision (v3.4), CastExpression grammar fix (v3.2), mode$ suffix fix (v3.2), stepXYZ! fix (v3.4)
- LSP specification: CompletionItem.tags, CompletionItemTag.Deprecated — official LSP 3.17 spec (verified HIGH confidence)
- Langium documentation: completion provider override patterns, cross-reference grammar rules — verified against Langium 4.1.3 behavior in codebase
- Bug reports: #379 (releaseVersion!), #380 (DECLARE in class), #381 (config.bbx), #382 (EM Config "--"), #373 (.class), #374 (static methods), #375 (SERIAL), #376 (EXIT int), #377 (ADDR)

---
*Pitfalls research for: BBj Language Server v3.9 — grammar additions, tokenizer bug fixes, and completion enhancements*
*Researched: 2026-02-20*
