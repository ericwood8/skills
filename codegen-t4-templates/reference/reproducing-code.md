# Reproducing hand-written code

Read this when: a template must produce what a person wrote by hand, or you are deciding whether generated output may overwrite a file.

## Reproducing existing code

1. Read **all** the hand-written examples and note what varies (hand-picked names, extra members, deliberate relaxations).
2. Read the live schema and data (read-only `sqlcmd`) for anything that becomes code (lookup rows become enum members).
3. Write the template so the output for the simplest example matches **exactly** (the diff may show only trailing whitespace, BOM, final newline). Project facts (namespaces, context type, enum tables, tables with no entity) are variables at the top of the template or project settings, never engine code.
4. Generate for **every** table into a scratch copy, compare with the originals (`git diff -w --ignore-cr-at-eol`). Overwrite only a close approximation; **revert the rest and list it as hand-maintained** with the reason.
5. Prove it: build the target. Generated code for tables with no screen or route is not type-checked unless referenced: route it in a scratch copy.
6. Fix a bug in the template so all tables get it, not only the sample.
7. **A generator that follows the column is stricter than hand code that relaxes it** (a nullable property over a NOT NULL column makes the API refuse requests that omit it). Treat "generated is stricter" as a reason to keep the file hand-maintained; "looser" is acceptable but must be reported.
8. A template header lists what a generator cannot know: collection navigation names, reverse navigations (both sides loop in JSON), enum-typed properties, validation attributes, hand-picked sort columns, detail grids and dependent drop-downs, registration lines.
