# ***Enhancing NuGet Audit for Secure and Selective `<PackageDownload>` Vulnerability Reporting***
<!-- Replace `Title` with an appropriate title for your design -->

- Author Name [Nigusu](https://github.com/Nigusu-Allehu)
- [GitHub Issue](https://github.com/NuGet/Home/issues/13658)

## Summary

<!-- One-paragraph description of the proposal. -->
In our effort to strengthen the security of NuGet, we would like to expand the support of NuGet Audit to `<PackageDownload>` packages. This will be done by introducing vulnerability errors for packages downloaded using `PackageDownload`. However, the .Net sdk also makes use of `PackageDownload` to implicitly download packages. And we would like to make sure these packages are not also receiving these errors, in order to prevent customers being confused. This will be done by introducing a new attribute to `PackageDownload` that makes it clear to NuGet Audit it should not write an error for these packages.

## Motivation 

<!-- Why are we doing this? What pain points does this solve? What is the expected outcome? -->

We are expanding NuGet Audit to `<PackageDownload>` packages to improve security notifications for user-specified dependencies, addressing gaps in vulnerability alerts that currently miss these packages. This change prevents confusion by excluding SDK-managed PackageDownloads from error reports, ensuring users only see relevant security issues. The expected outcome is a more precise and useful vulnerability reporting experience, enabling users to address actual risks without unnecessary SDK-related alerts.

## Explanation

### Functional explanation

<!-- Explain the proposal as if it were already implemented and you're teaching it to another person. -->
<!-- Introduce new concepts, functional designs with real life examples, and low-fidelity mockups or  pseudocode to show how this proposal would look. -->

The NuGet Audit now supports `<PackageDownload>` packages, allowing it to detect and alert users about vulnerabilities in any user-specified dependencies downloaded via `<PackageDownload>`. This means you’ll now receive warnings if a package added via `<PackageDownload>` has known security issues, so you can quickly address them.

However, the .NET SDK itself also uses `<PackageDownload>` to pull in SDK-specific dependencies, and to avoid confusing users, these SDK-managed packages are excluded from NuGet Audit warnings. We achieve this using a new attribute on `<PackageDownload>` entries in the SDK, such as `IsImplicitlyDefined="true"`. When NuGet Audit sees this attribute, it knows to skip auditing this package.

#### Real-Life Example

Imagine you have a project that downloads some libraries with `<PackageDownload>`:

```xml
<ItemGroup>
    <PackageDownload Include="NuGet.Protocol" Version="[5.11.2]" />
</ItemGroup>
```

If "NuGet.Protocol" is found to have a vulnerability, NuGet Audit will issue a warning to alert you so you can update to a safer version.

##### Warning NU1906

> warning NU1906: Package 'NuGet.Protocol' 5.11.2 has a known moderate severity vulnerability, https://github.com/advisories/GHSA-g3q9-xf95-8hp5


On the other hand imagine the SDK has the package `System.Text.Json`. The SDK would download this package as follows

```xml
<ItemGroup>
    <PackageDownload Include="System.Text.Json" Version="[1.0.0]" IsImplicitlyDefined="true"/>
</ItemGroup>
```

Since this was implicitly added, no warnings are received by the user.

#### Functional Design

Here’s a pseudocode version of how this exclusion works:

```csharp
foreach (package in PackageDownloads):
    if (package.HasAttribute("IsImplicitlyDefined") && package.IsImplicitlyDefined == true):
        continue  // Skip auditing, as it's an SDK-managed package
    else:
        checkForVulnerabilities(package)
```

Now this ensures users are provided with NuGet Audit warnings that are not confusing and actionable.

### Technical explanation

<!-- Explain the proposal in sufficient detail with implementation details, interaction models, and clarification of corner cases. -->

The following technical components will support the proposal's functionality:

1. **New Attribute for SDK-Specific PackageDownloads:** A new attribute, named `IsImplicitlyDefined="true"`, will be added to <PackageDownload> entries within the SDK. This will also be done in project-system to make sure VS nominations are working in a similar manner. Add the property [here](https://github.com/dotnet/project-system/blob/9f35656ad68aa1352d7b6b0fd01784f7aefe1005/src/Microsoft.VisualStudio.ProjectSystem.Managed/ProjectSystem/Rules/CollectedPackageDownload.xaml#L3) for project-system.

1. **NuGet Audit Update to Handle Attribute:** NuGet Audit will be modified to recognize the new attribute and exclude SDK-flagged PackageDownloads from security alerts. This update ensures that only relevant <PackageDownload> packages, added directly by the user, are scanned for vulnerabilities, allowing users to remain informed without SDK-related alerts.


## Drawbacks

<!-- Why should we not do this? -->

## Rationale and alternatives

<!-- Why is this the best design compared to other designs? -->
<!-- What other designs have been considered and why weren't they chosen? -->
<!-- What is the impact of not doing this? -->

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