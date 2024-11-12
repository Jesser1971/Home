# ***Enhancing NuGet Audit for Secure and Selective `<PackageDownload>` Vulnerability Reporting***
<!-- Replace `Title` with an appropriate title for your design -->

- Author Name [Nigusu](https://github.com/Nigusu-Allehu)
- [GitHub Issue](https://github.com/NuGet/Home/issues/13658)

## Summary

<!-- One-paragraph description of the proposal. -->
To improve the security of NuGet, we are expanding NuGet Audit's support to include `<PackageDownload>` packages, allowing it to report vulnerabilities for both user-specified and SDK-managed packages.
To prevent user confusion about SDK-included dependencies, we are introducing a new attribute, `AddedBy`, to identify the origin of each `<PackageDownload>`.
This will provide users with clear, actionable security information while helping them distinguish between SDK-provisioned packages and those they added manually.

## Motivation

<!-- Why are we doing this? What pain points does this solve? What is the expected outcome? -->

We are expanding NuGet Audit to `<PackageDownload>` packages to improve security notifications, addressing gaps in vulnerability alerts that currently miss these packages.
This change prevents confusion by clarifying SDK-managed PackageDownloads in the warning reports, ensuring users have an actionable warning.
The expected outcome is a more precise and useful vulnerability reporting experience, enabling users to address actual risks without confusing SDK-related alerts.

## Explanation

### Functional explanation

The NuGet Audit now supports `<PackageDownload>` packages, allowing it to detect and alert users about vulnerabilities in any user-specified dependencies downloaded via `<PackageDownload>`. 
This means you’ll now receive warnings if a package added via `<PackageDownload>` has known security issues, so you can quickly address them.

To prevent confusion about SDK-managed packages, we introduce a new attribute, `AddedBy`, on `<PackageDownload>` entries.
This attribute will specify the source of the package, with values like `"dotnetsdk"` for SDK-managed packages and `"user"` for user-added dependencies.
When a package marked with `AddedBy="dotnetsdk"` has a vulnerability, the warning will indicate that the package was added by the SDK.
This will help users understand that the package originates from the SDK and is not something they explicitly added.

| Attribute | Value     | Warning   |
|-----------|-----------|-----------|
| AddedBy   | dotnetsdk | warning NU1906: Package 'System.Text.Json' 1.0.0, added by '.NET SDK', has a known vulnerability. Refer to the vulnerability details at https://github.com/advisories/GHSA-g3q9-xf95-8hp5 to assess any necessary actions. |
| AddedBy   | user      | warning NU1906: Package 'System.Text.Json' 1.0.0, added by 'user', has a known vulnerability. Refer to the vulnerability details at https://github.com/advisories/GHSA-g3q9-xf95-8hp5 to assess any necessary actions. |
| AddedBy   | undefined | warning NU1906: Package 'System.Text.Json' 1.0.0 has a known vulnerability. Refer to the vulnerability details at https://github.com/advisories/GHSA-g3q9-xf95-8hp5 to assess any necessary actions. |

However, all of these warnings will be gated by the `SDKAnalysisLevel` property.
The warnings will only be shown if the user is using a .NET SDK version that introduced these warnings or newer.
For example, if these warnings are introduced in SDK version 9.0.300, the warnings will only appear if `SDKAnalysisLevel` is defined and set to 9.0.300 or higher.

This gating mechanism ensures that users will not see confusing or unexpected warnings for SDK-provided packages if they choose to configure their environment to use an older SDK version.
It allows NuGet Audit to provide clear, relevant security warnings without overwhelming users with alerts for packages controlled by earlier SDK versions.

#### Real-Life Example

Imagine you have a project that downloads some libraries with `<PackageDownload>`:

```xml
<ItemGroup>
    <PackageDownload Include="NuGet.Protocol" Version="[5.11.2]" />
</ItemGroup>
```

If "NuGet.Protocol" is found to have a vulnerability, NuGet Audit will issue a warning to alert you so you can update to a safer version.

##### Warning NU1906

> warning NU1906: Package 'System.Text.Json' 1.0.0 has a known vulnerability. Refer to the vulnerability details at https://github.com/advisories/GHSA-g3q9-xf95-8hp5 to assess any necessary actions

On the other hand imagine the SDK has the package `System.Text.Json`. The SDK would download this package as follows

```xml
<ItemGroup>
    <PackageDownload Include="System.Text.Json" Version="[1.0.0]" AddedBy="dotnetsdk"/>
</ItemGroup>
```

The warning for the following will make it clear that the package was added by the SDK

> warning NU1906: Package 'System.Text.Json' 1.0.0, added by '.NET SDK', has a known vulnerability. Refer to the vulnerability details at https://github.com/advisories/GHSA-g3q9-xf95-8hp5 to assess any necessary actions.


### Technical explanation

<!-- Explain the proposal in sufficient detail with implementation details, interaction models, and clarification of corner cases. -->

The following technical components will support the proposal's functionality:

1. **New Attribute for SDK-Specific PackageDownloads:** A new attribute, named `AddedBy="dotnetsdk"`, will be added to <PackageDownload> entries within the SDK.
 This will also be done in project-system to make sure VS nominations are working in a similar manner. 
 Add the property [here](https://github.com/dotnet/project-system/blob/9f35656ad68aa1352d7b6b0fd01784f7aefe1005/src/Microsoft.VisualStudio.ProjectSystem.Managed/ProjectSystem/Rules/CollectedPackageDownload.xaml#L3) for project-system.

1. **NuGet Audit Update to Handle Attribute:** NuGet Audit will be modified to recognize the new attribute and clarify SDK-flagged PackageDownloads in security alerts.
This update ensures that all <PackageDownload> packages are scanned for vulnerabilities, allowing users to remain informed.

## Drawbacks

<!-- Why should we not do this? -->

## Rationale and alternatives

<!-- Why is this the best design compared to other designs? -->
<!-- What other designs have been considered and why weren't they chosen? -->
<!-- What is the impact of not doing this? -->

Here are some alternate designs

- Warn for all `PackageDownload` packages without clarifying how the package was added.
  - This could lead to users getting confused.
    If a package downloaded by the .NET SDK has a vulnerability, the will end up getting a vulnerability warning.
    However, since the user did not add these packages, they could be lost on what action they should take in order to resolve the warnings.
- Warn only for user defined packages.
  - This prevents users from learning about their SDK being vulnerable.

## Prior Art

<!-- What prior art, both good and bad are related to this proposal? -->
<!-- Do other features exist in other ecosystems and what experience have their community had? -->
<!-- What lessons from other communities can we learn from? -->
<!-- Are there any resources that are relevant to this proposal? -->

## Unresolved Questions

<!-- What parts of the proposal do you expect to resolve before this gets accepted? -->
<!-- What parts of the proposal need to be resolved before the proposal is stabilized? -->
<!-- What related issues would you consider out of scope for this proposal but can be addressed in the future? -->

## Future Possibilities

<!-- What future possibilities can you think of that this proposal would help with? -->