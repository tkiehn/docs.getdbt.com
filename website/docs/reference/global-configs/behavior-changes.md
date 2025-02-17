---
title: "Behavior changes"
id: "behavior-changes"
sidebar: "Behavior changes"
---

Most flags exist to configure runtime behaviors with multiple valid choices. The right choice may vary based on the environment, user preference, or the specific invocation.

Another category of flags provides existing projects with a migration window for runtime behaviors that are changing in newer releases of dbt. These flags help us achieve a balance between these goals, which can otherwise be in tension, by:
- Providing a better, more sensible, and more consistent default behavior for new users/projects.
- Providing a migration window for existing users/projects &mdash; nothing changes overnight without warning.
- Providing maintainability of dbt software. Every fork in behavior requires additional testing & cognitive overhead that slows future development. These flags exist to facilitate migration from "current" to "better," not to stick around forever.

These flags go through three phases of development:
1. **Introduction (disabled by default):** dbt adds logic to support both 'old' and 'new' behaviors. The 'new' behavior is gated behind a flag, disabled by default, preserving the old behavior.
2. **Maturity (enabled by default):** The default value of the flag is switched, from `false` to `true`, enabling the new behavior by default. Users can preserve the 'old' behavior and opt out of the 'new' behavior by setting the flag to `false` in their projects. They may see deprecation warnings when they do so.
3. **Removal (generally enabled):** After marking the flag for deprecation, we remove it along with the 'old' behavior it supported from the dbt codebases. We aim to support most flags indefinitely, but we're not committed to supporting them forever. If we choose to remove a flag, we'll offer significant advance notice.

## What is a behavior change?

The same dbt project code and the same dbt commands return one result before the behavior change, and they return a different result after the behavior change.

Examples of behavior changes:
- dbt begins raising a validation _error_ that it didn't previously.
- dbt changes the signature of a built-in macro. Your project has a custom reimplementation of that macro. This could lead to errors, because your custom reimplementation will be passed arguments it cannot accept.
- A dbt adapter renames or removes a method that was previously available on the `{{ adapter }}` object in the dbt-Jinja context.
- dbt makes a breaking change to contracted metadata artifacts by deleting a required field, changing the name or type of an existing field, or removing the default value of an existing field ([README](https://github.com/dbt-labs/dbt-core/blob/37d382c8e768d1e72acd767e0afdcb1f0dc5e9c5/core/dbt/artifacts/README.md#breaking-changes)).
- dbt removes one of the fields from [structured logs](/reference/events-logging#structured-logging).

The following are **not** behavior changes:
- Fixing a bug where the previous behavior was defective, undesirable, or undocumented.
- dbt begins raising a _warning_ that it didn't previously.
- dbt updates the language of human-friendly messages in log events.
- dbt makes a non-breaking change to contracted metadata artifacts by adding a new field with a default, or deleting a field with a default ([README](https://github.com/dbt-labs/dbt-core/blob/37d382c8e768d1e72acd767e0afdcb1f0dc5e9c5/core/dbt/artifacts/README.md#non-breaking-changes)).

The vast majority of changes are not behavior changes. Because introducing these changes does not require any action on the part of users, they are included in continuous releases of dbt Cloud and patch releases of dbt Core.

By contrast, behavior change migrations happen slowly, over the course of months, facilitated by behavior change flags. The flags are loosely coupled to the specific dbt runtime version. By setting flags, users have control over opting in (and later opting out) of these changes.

## Behavior change flags

These flags _must_ be set in the `flags` dictionary in `dbt_project.yml`. They configure behaviors closely tied to project code, which means they should be defined in version control and modified through pull or merge requests, with the same testing and peer review.

The following example displays the current flags and their current default values in the latest dbt Cloud and dbt Core versions. To opt out of a specific behavior change, set the values of the flag to `False` in `dbt_project.yml`. You'll continue to see warnings for legacy behaviors that you have opted out of explicitly until you either resolve them (switch the flag to `True`) or choose to silence the warnings using the `warn_error_options.silence` flag.

<File name='dbt_project.yml'>

```yml
flags:
  require_explicit_package_overrides_for_builtin_materializations: False
  require_model_names_without_spaces: False
  source_freshness_run_project_hooks: False
```

</File>

When we use dbt Cloud in the following table, we're referring to accounts that have gone "[Versionless](/docs/dbt-versions/upgrade-dbt-version-in-cloud#versionless)."

| Flag                                                            | dbt Cloud: Intro | dbt Cloud: Maturity | dbt Core: Intro | dbt Core: Maturity | 
|-----------------------------------------------------------------|------------------|---------------------|-----------------|--------------------|
| require_explicit_package_overrides_for_builtin_materializations | 2024.04.141      | 2024.06.192         | 1.6.14, 1.7.14  | 1.8.0             |
| require_resource_names_without_spaces                           | 2024.05.146      | TBD*                | 1.8.0           | 1.9.0             |
| source_freshness_run_project_hooks                              | 2024.03.61       | TBD*                | 1.8.0           | 1.9.0             |
| [Redshift] restrict_direct_pg_catalog_access                    | 2024.09.242      | TBD*                | dbt-redshift v1.9.0           | 1.9.0             |

When the dbt Cloud Maturity is "TBD," it means we have not yet determined the exact date when these flags' default values will change. Affected users will see deprecation warnings in the meantime, and they will receive emails providing advance warning ahead of the maturity date. In the meantime, if you are seeing a deprecation warning, you can either:
- Migrate your project to support the new behavior, and then set the flag to `True` to stop seeing the warnings.
- Set the flag to `False`. You will continue to see warnings, and you will retain the legacy behavior even after the maturity date (when the default value changes).

###  Package override for built-in materialization 

Setting the `require_explicit_package_overrides_for_builtin_materializations` flag to `True` prevents this automatic override. 

We have deprecated the behavior where installed packages could override built-in materializations without your explicit opt-in. When this flag is set to `True`, a materialization defined in a package that matches the name of a built-in materialization will no longer be included in the search and resolution order. Unlike macros, materializations don't use the `search_order` defined in the project `dispatch` config.

The built-in materializations are `'view'`, `'table'`, `'incremental'`, `'materialized_view'` for models as well as `'test'`, `'unit'`, `'snapshot'`, `'seed'`, and `'clone'`.

You can still explicitly override built-in materializations, in favor of a materialization defined in a package, by reimplementing the built-in materialization in your root project and wrapping the package implementation.

<File name='macros/materialization_view.sql'>

```sql
{% materialization view, snowflake %}
  {{ return(my_installed_package_name.materialization_view_snowflake()) }}
{% endmaterialization %}
```

</File>

In the future, we may extend the project-level [`dispatch` configuration](/reference/project-configs/dispatch-config) to support a list of authorized packages for overriding built-in materialization.

<VersionBlock lastVersion="1.7">

The following flags were introduced in a future version of dbt Core. If you're still using an older version, then you have the legacy behavior by default (when each flag is `False`). 

</VersionBlock>

### No spaces in resource names

The `require_resource_names_without_spaces` flag enforces using resource names without spaces. 

The names of dbt resources (models, sources, etc) should contain letters, numbers, and underscores. We highly discourage the use of other characters, especially spaces. To that end, we have deprecated support for spaces in resource names. When the `require_resource_names_without_spaces` flag is set to `True`, dbt will raise an exception (instead of a deprecation warning) if it detects a space in a resource name.

<File name='models/model name with spaces.sql'>

```sql
-- This model file should be renamed to model_name_with_underscores.sql
```

</File>

### Project hooks with source freshness 

Set the `source_freshness_run_project_hooks` flag to `True` to include "project hooks" ([`on-run-start` / `on-run-end`](/reference/project-configs/on-run-start-on-run-end)) in the `dbt source freshness` command execution.

If you have specific project [`on-run-start` / `on-run-end`](/reference/project-configs/on-run-start-on-run-end) hooks that should not run before/after `source freshness` command, you can add a conditional check to those hooks:

<File name='dbt_project.yml'>

```yaml
on-run-start:
  - '{{ ... if flags.WHICH != 'freshness' }}'
```
</File>

## Adapter-specific behavior changes
Some adapters may show behavior changes when certain flags are enabled. Refer to the following sections for each respective adapter.
### [Redshift] restrict_direct_pg_catalog_access

Originally, the `dbt-redshift` adapter was built on top of the `dbt-postgres` adapter and used Postgres tables for metadata access. With this flag enabled, the adapter will use the Redshift API (through the Python client) if available, or query Redshift's `information_schema` tables instead of using `pg_` tables.

While we don't expect any user-noticeable behavior changes due to this change, out of caution we are gating it behind a behavior-change flag and encouraging users to test it before it becomes the default for everyone.
