# Stack Research

**Domain:** Langium-based BBj language server — grammar additions, parser bug fixes, completion enhancements
**Researched:** 2026-02-20
**Confidence:** HIGH (all findings from direct codebase inspection; no library changes required)

## Summary

All v3.9 features fit within the existing technology stack. No new npm packages, no Java library upgrades, no new Gradle dependencies. Every capability needed already exists in the installed versions of Langium 4.1.3, Chevrotain 11.0.3, and vscode-languageserver-types 3.17.5. The work is entirely in-codebase changes to grammar files, service classes, the java-interop backend, and one legacy `.cjs` command file.

## Recommended Stack

### Core Technologies (unchanged)

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Langium | 4.1.3 (locked) | Grammar DSL, AST generation, LSP plumbing | Grammar rule additions, scope changes, and completion provider changes are standard Langium operations — no upgrade needed |
| Chevrotain | 11.0.3 (locked) | Tokenizer underlying Langium | LONGER_ALT pattern (already used for 77 keywords) is the correct fix for #379 releaseVersion! identifier conflict |
| vscode-languageserver-types | 3.17.5 (transitive) | LSP protocol types including CompletionItemTag | `CompletionItemTag.Deprecated = 1` already exists at this version — deprecated indicator needs no library upgrade |
| vscode-languageserver | 9.0.1 (transitive) | LSP server runtime | No changes required |
| TypeScript | 5.8.3 | Language | Interface additions to java-types.langium and new TypeScript service methods |
| Vitest | 1.6.1 | Test runner | New parser/completion tests follow existing patterns exactly |

### Supporting Libraries (unchanged)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| langium-cli | 4.1.0 | Grammar code generation | Run `langium generate` after every `.langium` file edit — generates `generated/ast.ts` and `generated/grammar.ts` |
| vscode-jsonrpc | 8.2.1 | JSON-RPC transport for java-interop | No changes; existing socket connection reused |
| esbuild | 0.25.12 | Bundle LS for distribution | Run after TypeScript changes to produce `out/language/main.cjs` |

### Development Tools (unchanged)

| Tool | Purpose | Notes |
|------|---------|-------|
| `langium generate` | Regenerates `src/language/generated/ast.ts` from `.langium` files | Must run after any grammar edit; generated file must not be hand-edited |
| `vitest run` | Runs test suite | All 501 tests currently pass; new tests must maintain green baseline |
| `node ./esbuild.mjs` | Builds language server bundle | Required before IntelliJ plugin picks up changes |

## Installation

No new packages required.

```bash
# All dependencies already installed — no npm install needed for v3.9
# After grammar changes, regenerate AST:
npx langium generate
# After TypeScript changes, rebuild bundle:
node ./esbuild.mjs
```

## Feature-to-Stack Mapping

Each v3.9 feature maps to existing stack capabilities with specific integration points:

### Grammar verb additions (EXIT int, SERIAL)

**Files touched:** `bbj.langium`, `generated/ast.ts` (auto-regenerated)

- `EXIT int` (#376): Grammar rule `ExitWithNumberStatement` at line 473-474 has a precedence bug. Current form `kind='EXIT' | RELEASE_NL | RELEASE_NO_NL exitVal=Expression` means EXIT has no `exitVal`, only RELEASE does. Fix: change to `(kind='EXIT' exitVal=Expression?) | RELEASE_NL exitVal=Expression | RELEASE_NO_NL exitVal=Expression`. No new tokenizer custom rule needed — Expression handles integer literals already.

- `SERIAL` (#375): Not yet in `SingleStatement` list (confirmed TODO in VERBs.md). Add a `SerialStatement` rule. Pattern from BBj docs: `SERIAL fileid{,MODE=string}{,ERR=lineref}`. Follows same structure as `OpenStatement` without the channel number. Add to `SingleStatement` union; `SERIAL` keyword is non-reserved so `BBjTokenBuilder` LONGER_ALT handling (lines 39-49) already covers it automatically.

- `ADDR` (#377): Already implemented (grammar line 643, VERBs.md shows "OK"). Issue #377 may refer to a specific variant (e.g., `ADDR` as a function rather than verb) or a test coverage gap — verify against issue details before making changes.

### Parser bug fixes

**Files touched:** `bbj-token-builder.ts`, `Commands.cjs`, VS Code `package.json`

- `releaseVersion! parse error` (#379): `RELEASE_NL` and `RELEASE_NO_NL` are custom Chevrotain terminal tokens with negative-lookahead regexes (`/RELEASE(?!\s*(;\s*|\r?\n))/i`). When `releaseVersion!` is encountered, the regex matches the `RELEASE` prefix — the negative lookahead only excludes end-of-line, not word characters. Fix: tighten the regex to require that `RELEASE` is not followed by any word character: `/RELEASE(?![\w_])/i` for both tokens. This is safe because all legitimate `RELEASE expr` uses start with whitespace after RELEASE. The existing LONGER_ALT on the token (lines 52-57 of `bbj-token-builder.ts`) then ensures `releaseVersion!` tokenizes as `ID_WITH_SUFFIX` first.

- `DECLARE in class outside methods` (#380): `ClassMember` rule at line 346 is `FieldDecl | MethodDecl`. `VariableDecl` (`declare` statement) is not in `ClassMember`. Fix: add `VariableDecl` as a third alternative in `ClassMember`, or introduce a separate rule (`ClassVariableDecl`) that mirrors `VariableDecl`. Since `VariableDecl` is already defined (line 308-309), the grammar change is one line.

- `EM Config "--" startup failure` (#382): In `Commands.cjs` line 97, `configPath` is appended unconditionally to the `web.bbj` command string. When `configPath` is empty string, BBj receives `""` as the 10th argument which it interprets as a double-dash separator causing startup failure. The GUI run path at lines 229-230 already has the correct pattern: `const configArg = configPath ? \`-c"${configPath}" \` : ''`. Fix: apply the same conditional guard to the `runWeb` function's command string.

- `config.bbx not highlighted` (#381): The VS Code `package.json` registers only one language `"id": "bbj"` with extensions including `.bbx` (line 34). The `bbx.tmLanguage.json` grammar file exists in `syntaxes/` but is never registered as a grammar contribution in `package.json`. There is no `"language": "bbx"` grammar entry. Fix: add a second language entry for `bbx` in `package.json` with its own `id`, extension `.bbx`, and grammar pointing to `./syntaxes/bbx.tmLanguage.json`. Note: `.bbx` is also in the `bbj` language extensions — removing it from `bbj` and registering it under `bbx` may be the correct approach depending on whether `bbx` files should be parsed by the language server.

### Completion enhancements

**Files touched:** `java-types.langium`, `java-interop.ts`, `InteropService.java`, `MethodInfo.java`, `bbj-completion-provider.ts`, `bbj-type-inferer.ts`

- `Static method completion on class references` (#374): Requires end-to-end changes across three layers:
  1. **java-interop Java side** (`InteropService.java`, `MethodInfo.java`): Add `public boolean isStatic` to `MethodInfo`. In `loadClassInfo()`, set `mi.isStatic = Modifier.isStatic(m.getModifiers())`. `java.lang.reflect.Modifier` is already imported.
  2. **Langium schema** (`java-types.langium`): Add `isStatic?: boolean` to the `JavaMethod` interface.
  3. **Scope provider** (`bbj-scope.ts`): In `getScope()` for the `isMemberCall` branch, when `isJavaClass(receiverType)`, filter `receiverType.methods` to only `m.isStatic === true` when the receiver is a class reference (not an instance). The `BBjTypeInferer.getTypeInternal()` already returns the class node itself when `isClass(reference)` — this is the signal that the receiver is a class, not an instance.
  4. **Type inferer**: No change needed — `isClass` branch already works.

- `.class property` (#373): When `someVar!.class` is accessed, the `MemberCall` member resolves to nothing (`.class` is not a declared method). Fix in `bbj-type-inferer.ts` `getTypeInternal()`: add a check in the `isMemberCall` branch — if `expression.member.$refText.toLowerCase() === 'class'`, resolve to `javaInterop.getResolvedClass('java.lang.Class')`. No grammar change needed; `class` is a valid `ValidName` token already. No scope change needed; treat as a built-in pseudo-property.

- `Constructor completion` for `new ClassName()`: The grammar already has `ConstructorCall` rule (`bbj.langium` line 836-842). `BBjTypeInferer` already handles `isConstructorCall` (line 53-54). Completion enhancement needed in `BBjCompletionProvider`: when the cursor is preceded by `new `, offer all known `BbjClass` names (from scope) and imported `JavaClass` names with constructor-style snippet insertion. Implement by overriding `getCompletion()` to detect the `new ` prefix context, then enumerate classes from `importedClasses()` in `BbjScopeProvider`.

- `Deprecated method visual indicator`: `CompletionItemTag.Deprecated = 1` is confirmed in the installed `vscode-languageserver-types@3.17.5`. Two sub-parts:
  1. **Java methods**: Add `public boolean deprecated` to `MethodInfo.java`. Set via `m.isAnnotationPresent(Deprecated.class)` in `InteropService.java`. Add `deprecated?: boolean` to `JavaMethod` in `java-types.langium`. In `bbj-completion-provider.ts` `createReferenceCompletionItem()`, add `tags: [CompletionItemTag.Deprecated]` when the resolved node has `deprecated: true`.
  2. **BBj methods**: BBj has no native `@Deprecated` annotation. If the `DOCU` comment block contains `@deprecated`, parse it during class resolution in `java-interop.ts` and set a flag on the `MethodDecl`. This may be deferred — focus on Java method deprecated indicators first since those come from real Java reflection.

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| New npm packages for completion enhancements | Every capability (CompletionItemTag, scope APIs, type inferer) already exists in installed versions | Extend existing `BBjCompletionProvider` and `BbjScopeProvider` |
| Langium upgrade to 4.2+ | No features needed from newer versions; upgrade risk with no benefit | Stay on 4.1.3 |
| Separate custom tokenizer rules for SERIAL | BBjTokenBuilder's LONGER_ALT pattern handles all uppercase non-reserved keywords uniformly | Add SERIAL as a plain keyword in the grammar; it gets LONGER_ALT automatically |
| Modifying `generated/ast.ts` directly | Auto-generated file; changes are overwritten by `langium generate` | Edit only `.langium` grammar files, then regenerate |
| Adding `static` filtering at the BBj class level | BBj `STATIC` methods are valid instance-accessible in some contexts; filtering would break existing behavior | Filter `isStatic` only in the Java class member scope path (`isJavaClass(receiverType)` branch) |
| Custom LSP extension for deprecated | Standard `CompletionItemTag.Deprecated = 1` is rendered by both VS Code and LSP4IJ as strikethrough | Use standard LSP tags array |

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| langium@4.1.3 | chevrotain@11.0.3 | Langium 4.1.x ships this exact Chevrotain version as peer dep |
| vscode-languageserver@9.0.1 | vscode-languageserver-types@3.17.5 | Transitive dep; CompletionItemTag.Deprecated is at 3.17.x |
| java-interop MethodInfo (new `isStatic` field) | vscode-jsonrpc@8.2.1 | JSON-RPC serialization handles new fields transparently; old clients ignore unknown fields |
| java-interop MethodInfo (new `deprecated` field) | existing java-interop connection protocol | Additive field — backward compatible with older java-interop instances that omit it |
| `CompletionItemTag` import | vscode-languageserver@9.0.1 | Already available via `vscode-languageserver` — same import path as `CompletionItemKind` |

## Integration Workflow Summary

### Grammar change (every grammar feature)
1. Edit `bbj.langium` (or `java-types.langium` for type schema)
2. Run `npx langium generate` — regenerates `src/language/generated/ast.ts`
3. Fix TypeScript compilation errors in dependent service files
4. Add/update Vitest tests in `test/parser.test.ts`
5. Run `node ./esbuild.mjs` to rebuild the LS bundle

### java-interop change (static methods, deprecated)
1. Edit `MethodInfo.java` (add field)
2. Edit `InteropService.java` (populate field via reflection)
3. Edit `java-types.langium` (add field to `JavaMethod` interface)
4. Run `npx langium generate` (regenerates AST types)
5. Update `java-interop.ts` to read and propagate new field from JSON response
6. Update `bbj-completion-provider.ts` to use new field
7. Build java-interop JAR and rebuild LS bundle

### Completion provider change (no java-interop needed)
- `.class` property and constructor completion: modify only `bbj-type-inferer.ts` and `bbj-completion-provider.ts`
- No grammar changes required for these two features

## Sources

- Direct inspection of `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj.langium` — grammar rules, ExitWithNumberStatement, SERIAL status, ClassMember definition (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/bbj-token-builder.ts` — LONGER_ALT pattern and RELEASE token regexes (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/java-interop/src/main/java/bbj/interop/data/MethodInfo.java` — current MethodInfo fields (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/java-interop/src/main/java/bbj/interop/InteropService.java` — `java.lang.reflect.Modifier` already imported, `m.getModifiers()` pattern available (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/bbj-vscode/node_modules/vscode-languageserver-types/lib/esm/main.js` line 1144 — `CompletionItemTag.Deprecated = 1` confirmed present (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/Commands/Commands.cjs` — EM config "--" root cause confirmed at line 97 (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/bbj-vscode/package.json` — missing bbx grammar registration confirmed (HIGH confidence)
- `/Users/beff/_workspace/bbj-language-server/bbj-vscode/VERBs.md` — SERIAL confirmed TODO, ADDR confirmed OK (HIGH confidence)
- Direct inspection of `/Users/beff/_workspace/bbj-language-server/bbj-vscode/src/language/java-types.langium` — current JavaMethod interface (no deprecated, no isStatic) (HIGH confidence)

---
*Stack research for: BBj language server v3.9 — grammar additions, bug fixes, completion enhancements*
*Researched: 2026-02-20*
