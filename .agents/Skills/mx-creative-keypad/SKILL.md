---
name: mx-creative-keypad
description: Create, inspect, modify, and validate Logitech MX Creative Console LP5 profiles and C# plugins for the Keypad or Dialpad. Use when a user requests an .lp5 profile, a C# .lplug4 plugin, Logi Options+ actions, keypad pages, dial actions, dynamic folders, keyboard or text macros, application binding, service integration, plugin packaging, or SVG action icons.
license: MIT
compatibility: LP5 work requires tools that can read and create ZIP archives and JSON files. C# plugin work requires the .NET 8 SDK, LogiPluginTool, and Logi Options+ or Loupedeck with Logi Plugin Service. Use current official SDK documentation and current local exports as format references.
metadata:
  version: "1.1.0"
---

# MX Creative Keypad Profiles and C# Plugins

Create importable `.lp5` profiles and C# `.lplug4` plugins for Logi Options+. Treat both formats as separate artifacts with different capabilities.

## Choose the artifact type

- Use an `.lp5` profile for fixed pages, existing actions, keyboard shortcuts, text, and macros.
- Use a C# `.lplug4` plugin for custom commands, dial adjustments, live state, external application or service APIs, and plugin-controlled dynamic folders.
- A C# application plugin can include default `.lp5` profiles in its `profiles` package folder.
- Do not use hand-authored LP5 records to imitate behavior that needs runtime state. Implement that behavior in a plugin.

## Required inputs

Identify these values before generation:

- Artifact type: standalone `.lp5` profile or C# `.lplug4` plugin.
- Target device: MX Creative Keypad, Dialpad, or another supported device.
- Target application and its process or bundle name.
- Page names, control positions, and action behavior.
- Output path and profile display name.
- A current exported `.lp5` example when one is available.
- Icon style, palette, and label requirements.
- For plugins: unique plugin name, display name, author, version, license, supported operating systems, and application or service integration.

Ask the user only when a missing choice changes the layout or action behavior. Inspect the local Logi profile data when it can answer the question safely.

## LP5 profile workflow

1. Inspect repository and workspace instructions before changing files.
2. Inspect the current Git status when the output is inside a repository.
3. Read the supplied LP5 example as a ZIP archive. Do not edit the example.
4. Read [the LP5 format reference](references/lp5-format.md).
5. Confirm the device type from a current export or installed Logi profile. Do not guess an unknown device type.
6. Create an action map before creating files. Record every page, control index, action identifier, label, and icon.
7. Keep standalone profiles independent from native plugins unless the user explicitly requests a plugin-backed profile.
8. Generate package files in a task-specific `.temp/` directory.
9. Generate one clear SVG icon for each action. Keep a copy of each SVG file in the project-root `./icons/` folder. Create the folder when it does not exist. Embed each SVG as base64 in its matching `.ict` action icon template.
10. Create the ZIP archive with the package files at the archive root. Rename the final ZIP file to `.lp5`.
11. Validate the archive, JSON, actions, and SVG content before delivery.
12. Delete only the task-specific temporary directory after validation.

## Action design

- Bind a standalone application profile to the exact process or bundle name.
- Use generic keyboard, text, application, and macro actions that exist in the target Logi installation.
- Copy keyboard layout identifiers and serialized key formats from a current local export. Do not invent them.
- Use uppercase 32-character hexadecimal identifiers for profile, workspace, page, macro, and editor-action names.
- Keep identifiers unique inside the profile. Deterministic identifiers are acceptable when the generator uses stable input names.
- Keep control order explicit. The control index defines the physical position.
- For long text actions, use short key labels and store the full text in the macro action.
- Use navigation controls for approval menus. Do not create an automatic approval action unless the target session and pending request can be verified.

## Page navigation and simulated submenus

Treat an LP5 profile as a flat set of pages. Do not assume that it supports a nested menu hierarchy. Use ordinary pages to simulate submenus:

1. Create a main page.
2. Create a separate page for each submenu.
3. On the main page, assign the MX Creative Keypad `Navigate to page` action to a control and select the submenu page.
4. On the submenu page, assign another `Navigate to page` action to a Back control and select the main page.

The submenu page is still a normal profile page. The physical page-left and page-right controls can reach it. The profile does not store a parent-child relationship for this simulated submenu. The Back control is a direct page-navigation action, not navigation history.

The public Logi Actions SDK documentation describes LP5 profile packages, but it does not define the serialized record for the `Navigate to page` action. Copy the complete action identifier and parameters from a current export. Do not invent them. If no suitable export is available, create the pages and navigation actions in Logi Options+, export the LP5 profile, and use that export as the serialization reference.

See the [Logitech community page-navigation guidance](https://www.reddit.com/r/logitech/comments/1hjescf/mx_creative_console_questions_and_feature_requests/) and [Logi Actions SDK plugin basics](https://logitech.github.io/actions-sdk-docs/plugin-basics/).

Use a C# dynamic folder when the content or hierarchy must change at runtime. A dynamic folder is controlled by the plugin and has its own Back and page-navigation behavior.

## C# plugin workflow

1. Read the current [Logi Actions SDK C# documentation](https://logitech.github.io/actions-sdk-docs/csharp/) before selecting SDK types or package fields.
2. Confirm that Logi Options+ or Loupedeck, Logi Plugin Service, and the .NET 8 SDK are installed.
3. Check for `LogiPluginTool`. Ask before installing a missing global tool.
4. Generate a current skeleton with `logiplugintool generate <Name>`. Prefer this skeleton over an old third-party template.
5. Implement `{Name}Plugin : Plugin` for plugin-level behavior and `{Name}Application : ClientApplication` for application integration.
6. Implement commands with `PluginDynamicCommand`, dial adjustments with `PluginDynamicAdjustment`, and plugin-controlled workspaces with `PluginDynamicFolder`.
7. Keep device actions thin. Put application APIs, network access, state management, and other business logic in focused services behind interfaces.
8. Configure `metadata/LoupedeckPackage.yaml`, the plugin icon, supported devices, supported operating systems, binary folders, and optional default profiles.
9. Add xUnit tests for business logic, parsing, state changes, and error paths. Isolate the SDK boundary so that tests do not require a connected device.
10. Build and test the solution. The generated project creates a `.link` file that lets Logi Plugin Service load the development output.
11. Test the actions on each supported control type and device. Inspect the plugin log for errors.
12. Build Release, create the `.lplug4` package with `logiplugintool pack`, and validate it with `logiplugintool verify`.

Use these commands as the baseline. Replace placeholders and use the actual Release output directory:

```text
logiplugintool generate <Name>
dotnet build
dotnet test
logiplugintool pack <release-output-directory> <plugin-name>_<version>.lplug4
logiplugintool verify <plugin-name>_<version>.lplug4
```

## C# plugin action design

- Use `PluginDynamicCommand` for one discrete button action.
- Use `PluginDynamicAdjustment` for dial rotation. Implement reset behavior only when a press has a clear reset meaning.
- Use `PluginDynamicFolder` for a runtime folder or control center. The plugin owns all content in this workspace.
- Keep dynamic-folder constructors limited to metadata. Use `Load` and `Unload` for plugin lifetime, and `Activate` and `Deactivate` for subscriptions needed only while the folder is visible.
- Select a `PluginDynamicFolderNavigation` mode. Use `NavigateUpActionName`, `NavigateLeftActionName`, and `NavigateRightActionName` when the plugin supplies navigation controls itself.
- Notify Logi Plugin Service when action lists, images, labels, or adjustment values change.
- Use `###` in an action `groupName` only to create action-picker groups. The SDK supports at most three group levels. These groups are not device submenus.
- Keep network and file operations asynchronous where the SDK permits it. Handle cancellation, timeouts, retries, and disconnected services.
- Store credentials with the plugin settings or data facilities. Do not put secrets in source files, package metadata, default profiles, or logs.

## C# plugin package design

- Set `type: plugin4` in `metadata/LoupedeckPackage.yaml`.
- Select a stable plugin `name`. Do not end it with `Plugin`; do not change it after publication.
- Put Windows binaries in the configured `win` folder and macOS binaries in the configured `mac` folder. Include only the operating systems that the package supports.
- Put the 256-pixel plugin icon and package metadata in `metadata`.
- Put automatic action images in `actionicons`, icon templates in `icontemplates`, action-picker symbols in `actionsymbols`, and default LP5 profiles in `profiles`.
- For an MX Creative Keypad default profile, export it from Logi Options+ as `DefaultProfile70.lp5`. Check the export for personal information before packaging it.
- Keep package versions and assembly versions aligned.

## SVG icon design

- Use a `64 x 64` or `32 x 32` view box.
- Encode SVG files as UTF-8 without a byte-order mark (BOM). Logi Options+ does not render embedded SVG payloads that start with the UTF-8 BOM bytes `EF BB BF`.
- Use only colors from [the Material Design color palette](references/MaterialColors.md) in SVG files.
- Use simple paths, shapes, and short text that remain clear on a small LCD key.
- Use one semantic accent color per page or action group.
- Use error colors only for destructive actions such as Stop or Exit.
- Keep strong contrast between the background and the glyph.
- Put the full action label in the `.ict` text item. Keep the SVG glyph concise.
- Create a unique SVG payload for every action, even when two actions use similar symbols.
- Do not add separate SVG files to the LP5 archive unless a current exported profile uses them.
- For plugins, follow the SDK naming rules for `actionicons`, `icontemplates`, and `actionsymbols` so Logi Plugin Service can discover each asset.

## LP5 validation

Verify all of these conditions:

- The final file starts with the ZIP signature `50 4B 03 04`.
- `ApplicationInfo.json`, `ProfileInfo.json`, and `metadata/LoupedeckPackage.yaml` exist at the archive root.
- All JSON files parse successfully.
- The manifest name matches the profile name.
- The application and profile device types match.
- Every layout control action resolves to a defined macro, profile action, or installed native action.
- Every custom action has the expected `.ict` file.
- Every embedded base64 image decodes successfully.
- Every source and decoded SVG uses UTF-8 without a BOM.
- Every embedded SVG parses as XML.
- Every embedded SVG has a matching source copy in the project-root `./icons/` folder.
- Page and control counts match the requested hardware layout.
- The archive contains no temporary files or source-only content.
- The original example and installed Logi profile data are unchanged.

If Logi Options+ import testing is allowed, import the new profile only after the structural checks pass. Do not replace an existing customized profile without explicit permission.

## C# plugin validation

Verify all of these conditions:

- Restore, build, and xUnit tests pass in the intended configuration.
- `metadata/LoupedeckPackage.yaml` has `type: plugin4`, a stable name, a valid version, author, license, license URL, support URL, plugin file name, and correct operating-system folders.
- The package contains the declared binaries, plugin icon, action assets, and optional profiles and localization files.
- The target device and host software can discover the plugin through the development `.link` file.
- Commands run once per activation, adjustments handle positive and negative ticks, and reset behavior is correct.
- Dynamic folders show the correct controls, paginate correctly, update live state, and provide a working Back path.
- Application changes, device disconnects, service failures, timeouts, and malformed responses do not crash Logi Plugin Service.
- Plugin logs contain enough diagnostic context and contain no credentials or personal data.
- `logiplugintool verify` accepts the final `.lplug4` package.
- A hardware test confirms each claimed device-specific behavior. Report package verification and hardware testing as separate results.

Install the `.lplug4` package only when the user allows it. Do not replace an installed development or customized plugin without explicit permission.

## Delivery

Report:

- The final `.lp5` or `.lplug4` path.
- The target device and application binding.
- For a plugin: plugin name, version, supported operating systems, action types, dynamic-folder count, and external integrations.
- The page, action, and icon counts.
- Build, test, package-verification, host-test, and hardware-test results. State clearly when a test was not run.
- Any behavior that requires the target application to be active.
- Any feature that a static LP5 profile cannot provide, such as live session state or verified session targeting.

Do not commit, push, install, or import a profile or plugin unless the user requests that action.

## C# plugin references

- Use the [Logi Actions SDK C# documentation](https://logitech.github.io/actions-sdk-docs/csharp/) as the primary reference.
- Follow the official guidance for [plugin structure](https://logitech.github.io/actions-sdk-docs/csharp/tutorial/plugin-structure/), [dynamic folders](https://logitech.github.io/actions-sdk-docs/csharp/plugin-features/implementing-dynamic-folders/), [default profiles](https://logitech.github.io/actions-sdk-docs/csharp/plugin-features/default-application-profiles/), and [plugin distribution](https://logitech.github.io/actions-sdk-docs/csharp/plugin-development/distributing-the-plugin/).
- Use Logitech's [Home Assistant Options+ plugin](https://github.com/Logitech/cto-HomeAssistantPlugin-OptionsPlus/) as a working C# example for services, tests, dynamic folders, packaging, and release validation.
- The [XeroxDev Loupedeck plugin template](https://github.com/XeroxDev/Loupedeck-plugin-Template) is an archived third-party template. Use it only as a secondary reference. Prefer a skeleton generated by the current Logi Plugin Tool.
