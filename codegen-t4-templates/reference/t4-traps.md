# T4 and escape traps

Read this when: a template fails to compile, output has wrong line endings or quotes, or you are editing a template that writes strings with backslashes.

## T4 and escape traps

- Signal an error with `Error("...")` in a statement block and wrap **all output in the success branch**. A compile error comes back in `TemplateResult.Errors`: put `string.Join(" | ", result.Errors)` in test assertion messages. Usual causes: missing `<#@ import namespace="System.Data" #>` (for `SqlDbType`) or `System.Collections.Generic` (for `List<>`).
- Build output with a `StringBuilder` and `Append('\n')` (`Environment.NewLine` mixes CRLF into LF). Templates are LF in the repo, CRLF in a Windows checkout: normalise `\r\n` in test helpers. Generated files have no BOM.
- A template gets `Model` (`TableModel`) and `Project`; extra needs are `.tt.config` switches: `NeedsRowData`, `NeedsReferencedDisplayColumns`, `RequiresPrimaryKey`, `TableOnly`, `OutputName` (a pattern with `{Table}`).
- Templates are **embedded resources**: rebuild the CLI after editing; the exe is in `bin\Debug\net10.0\`, not an older `win-x64\` folder. The CLI seeds its `Templates` folder once and never overwrites it: **delete `bin\Debug\net10.0\Templates` before a run** or old templates run and look like a template bug.
- **Never write `--` in template text** (illegal in a generated XML/XAML/HTML comment; a test scans `Templates\*.tt`). Use ` - `, a colon, `=====`. Legitimate: `<!--`/`-->`, an SQL comment at the start of a line in an `SP_` template, C# `i--`. For a CLI flag in generated text build it: `new string((char)45, 2) + "port"`; name a flag in words in a comment. Markdown table separator rows and Mermaid arrows need hyphens: `new string((char)45, n)`.
- `global::Ns.Type` inside `{ }` of an interpolated string reads `::` as a format specifier: use a variable.
- `ColumnModel.MaxLength` of an `nvarchar` is in **bytes**: halve it for characters.
- Reserved-word names come from the schema reader (`IsCSharpReservedWordName`); `Sample.Column` in tests sets it from the name.
- A word ending in `s` is not pluralised (`CustomerStatus` is `/api/customerstatus`): take plurals from `Pluralizer`, never guess them in a test.
- **Editing a template that writes C#/TypeScript strings with `\n`, quotes or backslashes:** use the Edit tool, or a script file made with the Write tool that does exact-string replace and **throws unless the match count is exactly 1** (Node: `s.replace(a, () => b)` so `$` is not interpreted). A `\n` meant for the generated file written through a Python or shell string became a real line break (CS1010 "Newline in constant"). `grep` the changed line afterwards. See the git-bash-editing skill. Test helpers that render a template should build keys with `FromValues`, not strings with line breaks.
- To emit lines from a T4 helper: `void Emit(List<string> lines)` calling `Write(line)` and `Write(((char)10).ToString())`.
- To see what the schema reader decided, write a throwaway console project referencing Core, Connections and SchemaIntrospection and print the model's flags; a diagnostic `.tt` through the CLI gives only a stack trace.

## Line endings after a checkout

- After `git checkout`, templates are CRLF again: a script that edits them normalises `\r\n` first and asserts exactly one match.
