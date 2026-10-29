# Plan, template shapes and adding a template or setting

Read this when: you add or change a template, a .tt.config key, a project setting, or decide whether a template fits only some tables.

## Plan, shapes and adding things

**Template shapes:** per table (`Model`, default; `RequiresPrimaryKey`, `TableOnly`, `OutputName`); `DatabaseOnly=true` (`Database`, one file for the whole database, an `@@@FILE` marker; the test that renders every shipped template feeds it a `DatabaseModel`); `NoDatabase=true` (only `Project`; run by hand or an essentials group); `InPlan=false` + the project's `PlanAlso` (opt-in); `Dialects=` (skipped quietly for another database); `AccessMode=Routines|Ef`. `SqlServerOnly=true` is forbidden by a test: use `Dialects=` and, for a per-table template run by hand, an `Error(...)` on `Model.Dialect`. Shared front-end support code (grid sort helpers) is a database-level template with literal text after an `@@@FILE` marker, not a hand-copied file.

**Whole-project plan keys:** `Stacks=`, `PlanTables=` (Entity, Context, Api, AspNetApi, Search, Screen, ScreenForm, ScreenDetailMaster, Junction, Enum, Audit, Temporal), `OutputRoot=`, `OutputFolder[.<Stack>]=`, `InPlan=false`, `Dialects=`. A template without them is silently left out. Compile each template once per run (`TemplateCache`); re-set session values and clear `Errors` between runs.
**Adding a template:**
1. `Templates/<Group>_<Name>_v1.tt` + `.tt.config`; the prefix picks the menu and default extension; a **new prefix** needs `TemplateInfo.ExtensionByGroup`.
2. `DefaultAssetSeeder.DefaultTemplateFileNames` (both files), or it is not shipped.
3. Group counts in `TemplateEngineTests.The_shipped_templates_are_all_offered_with_their_groups`.
4. `Docs/TemplateNotes/<Name>_v1.md`, the README row and count, a settings hint and key for any new setting.
5. A template that a project setting switches on (`ApiCsv`, the gates) is also listed in `ProjectSettings.ImpliedPlanTemplates`, for every stack that has one (`API_Csv`, `PY_Csv`, `RS_Csv`), besides `InPlan=false`. A template with `PlanTables=Junction` is in the plan of a project that has no junction table (it just runs for no table): the plan-list tests (`RustTests`, `PythonTests`) name it.
6. A rendering test, and for generated **code** a real compile (several tables of a sample into a scratch project with a small round trip); for SQL apply it to a scratch database copy.

**Adding a project setting:** the `Keys` list, a one-line hint in `ProjectSettingsHints` (effect and what blank means; do not start with the value to type), a row in `Docs/Reference.md`, a tab in `ProjectSettingGroups` (at most 20 per tab, every key on exactly one tab), the control (`BooleanKeys`, `ProjectSettingChoices`, its `Numbers` table with min 1; a radio has at most 3 choices). A flag that adds templates (`ImpliedPlanTemplates`) says in its hint that the project must be regenerated and which stacks it needs. Optional features default off. When removing a setting remove it from all of those places plus the template header, notes, README and specs.
**A template that only fits some tables:** derive it from the table, not a setting. Put the shape rule in Core once (`AuditTableShape` used by `TableModel.IsAuditTable` and `AuditColumnClassifier`). A new "requires" key needs the property and parse in `TemplateConfig`, a reason in `TemplateConfig.Refuse`, a parameter on `TemplateInfo.AppliesTo`, the flag on `TableSummary` computed in **every** provider's bulk query, the call in `MainViewModel.GetApplicableTemplates`, and a `Reference.md` row. Keep it out of the default plan with `InPlan=false`. A template that writes a second table refuses clashes up front with a reason.
**Optional extras are switches:** a flag mapped by `ImpliedPlanTemplates` (`ApiDocs`, `ApiHttp`, `ApiFakers`, `ProjectDocs`, `ApiValidation`) or a branch in an essentials/template body, **off by default**, tested in both states.
