# Markdown README Rendering in PM UI

- Jonatan Gonzalez ([jgonz120](https://github.com/jgonz120)) 
- Issue [#12583](https://github.com/NuGet/Home/issues/12583) <!-- GitHub Issue link -->
- [Feature Spec](https://github.com/NuGet/Home/blob/7943122dffa435f4daeee600efcc5b744cd2e97e/accepted/2023/PMUI-README-rendering.md)

## Summary

We want to update the packages pane in the PMUI to render the README for the selected package. 
We also need to update the Nuget Server API to provide a link to download the README directly, without having to download the nupkg first.

## Motivation 

The README file is an essential source of information that can help customers understand what a package does.
By rendering the README file directly in the PM UI, customers can easily access the information they need to determine if a package will be helpful to them while searching, browsing, and managing their projects’ packages.
This enhancement streamlines the user experience and may encourage package authors to create comprehensive README documentation when customers will be more likely to rely on it for useful information about the package.

## Explanation
### Functional explanation
#### PM UI
![Alt text](../../meta/resources/ReadMePMUI/ReadmeTab.png)
![Alt text](../../meta/resources/ReadMePMUI/NoReadmeTab.png)

The PM UI will be updated to have tabs for the Package Details and the the README.
This UX will be displayed for both the Browse and installed tabs.
It will also be displayed for both the solution level and project level package managers.

##### README Tab
This tab is rendered if a remote source available to download the readme or the Installed tab is selected and the selected version is found in the global packages folder.
This is to ensure that users using the Browse tab will not see the README tab if they do not have a remote source that allows README downloads.

If the README tab is rendered and we do not find a README then a message is rendered encouraging the user to contact the package author to upload a README.
![alt text](../../meta/resources/ReadMePMUI/NoReadMeFound.png)

##### Package Details
This tab is always rendered and contains the package details information along with the Vulnerabilty and Depreciation information. 

##### README File Sources

* ReadmeFileUrl in the [package metadata](https://learn.microsoft.com/en-us/nuget/api/registration-base-url-resource).
* README direct download specified in a new resource `ReadmeUriTemplate`.
* Downloaded nupkg.

#### NuGet API
##### New RegistrationsBaseUrl version

A new version of the [package metadata](https://learn.microsoft.com/en-us/nuget/api/registration-base-url-resource) resource type will be documented which will include the field **ReadmeFileUrl**.
This will be a link to download the README and will only be filled if a readme is available to download.

###### index.json

```json
{
    "version": "3.0.0",
    "resources": [
        ...,
        {
            "@id": "https://apidev.nugettest.org/v3/registration5-gz-semver2/",
            "@type": "RegistrationsBaseUrl/6.13.0"
        },
        ...
    ]
}
```
###### Example Response
```json
{
    "@id": "https://apidev.nugettest.org/v3/registration5-gz-semver2/newtonsoft.json/index.json",
    "items": [
        {
            ...,
            "items": [ 
                ...,
                {
                    "@id": "https://apidev.nugettest.org/v3/registration5-gz-semver2/newtonsoft.json/13.0.3.json",
                    "@type": "Package",
                    "catalogEntry": {
                        "@id": "https://apidev.nugettest.org/v3/catalog0/data/2024.08.19.16.55.59/newtonsoft.json.13.0.3.json",
                        "@type": "PackageDetails",
                        "authors": "James Newton-King",
                        "dependencyGroups": [ ... ],
                        "description": "Json.NET is a popular high-performance JSON framework for .NET",
                        "iconUrl": "https://apidev.nugettest.org/v3-flatcontainer/newtonsoft.json/13.0.3/icon",
                        "id": "Newtonsoft.Json",
                        "language": "",
                        "licenseExpression": "MIT",
                        "licenseUrl": "https://dev.nugettest.org/packages/Newtonsoft.Json/13.0.3/license",
                        "readmeUrl": "https://dev.nugettest.org/packages/Newtonsoft.Json/13.0.3#show-readme-container",
                        //New field
                        "readmeFileUrl": "https://apidev.nugettest.org/v3/flatcontainer/newtonsoft.json/13.0.3/readme",
                        "listed": true,
                        "minClientVersion": "2.12",
                        "packageContent": "https://apidev.nugettest.org/v3-flatcontainer/newtonsoft.json/13.0.3/newtonsoft.json.13.0.3.nupkg",
                        "projectUrl": "https://www.newtonsoft.com/json",
                        "published": "2023-04-25T14:48:53.817+00:00",
                        "requireLicenseAcceptance": false,
                        "summary": "",
                        "tags": [
                            "json"
                        ],
                        "title": "Json.NET",
                        "version": "13.0.3"
                    },
                    "packageContent": "https://apidev.nugettest.org/v3-flatcontainer/newtonsoft.json/13.0.3/newtonsoft.json.13.0.3.nupkg",
                    "registration": "https://apidev.nugettest.org/v3/registration5-gz-semver2/newtonsoft.json/index.json"
                },
                ...
            ],
            ...
        }
    ],
    ...
}
```
##### ReadmeUriTemplate Resource Type
A new resource `ReadmeUriTemplate` similar to the [ReportAbuseUriTemplate](https://learn.microsoft.com/en-us/nuget/api/report-abuse-resource) resource type which will include a url definition for downloading the README.

```json
{
    "version": "3.0.0",
    "resources": [
        ...,
        {
            "@id": "https://apidev.nugettest.org/v3/flatcontainer/{lower_id}/{lower_version}/readme",
            "@type": "ReadmeUriTemplate/6.13.0"
        },
        ...
    ]
}
```
In code we would take the id provided and replace the {lower_id} and {lower_version} in the strings with the package we are trying to get the README for.

### Technical explanation
#### Rendering Markdown

We will use the [Microsoft.VisualStudio.Markdown.Platform](https://dev.azure.com/azure-public/vside/_artifacts/feed/vs-impl/NuGet/Microsoft.VisualStudio.Markdown.Platform/overview/18.0.39-preview-g659b28ccd8) package to render the README in the IDE. This will allow us to leverage a centralized tool for rendering markdown in the IDE. 

A new instance of the [IMarkdownPreview](https://devdiv.visualstudio.com/DevDiv/_git/VS-Platform?path=%2Fsrc%2FProductivity%2FMarkdownLanguageService%2FImpl%2FMarkdown.Platform%2FPreview%2FIMarkdownPreview.cs) can be created using [PreviewBuilder](https://devdiv.visualstudio.com/DevDiv/_git/VS-Platform?path=/src/Productivity/MarkdownLanguageService/Impl/Markdown.Platform/Preview/PreviewBuilder.cs).

We can use the Preview builder as follows:
```C#
//This creates a new instance of the preview builder
var markdownPreview = new PreviewBuilder().Build();

//We update the current markdown being rendered by calling "UpdateContentAsync"
markdownPreview.UpdateContentAsync(markDown ?? string.Empty, ScrollHint.None)

//IMarkdownPreview.VisualElement contains the FrameworkElement to be passed to the view
MarkdownPreviewControl = markdownPreview.VisualElement
```
#### Locating the README

Create a new implementation of the `INuGetResource` interface, `ReadmeUriTemplateResource`.
This will only be available for sources which have implemented the new `ReadmeUriTemplate` resource type.

Update `IPackageSearchMetadata` to include the `ReadmeFileUrl` field. 
`LocalPackageSearchMetadata` will populate the field with the location of the README, if available, on the local disk.
When a server implements the new `RegistrationsBaseUrl` resource type, then the `ReadmeFileUrl` field will be returned from the server if a README is available and be deserialized from the response.
When a server implements the `ReadmeUriTemplate` resource type and `ReadmeFileUrl` is empty, it will be populated with the url to the README. 

#### Downloading the README

Update the `INuGetPackageFileService` to add a new method for downloading the README. 

## Drawbacks

MarkdownPreview control currently marked as obsolete since the interface has not been finalized.
So when an upgrade is made we may have to change how we use the control.

WebView2 controls always render ontop of other controls in the view.
[Secnario 25254665](https://microsoft.visualstudio.com/Edge/_workitems/edit/25254665).
PM UI needs to be updated to ensure items don't scroll off screen.

`INuGetPackageFileService` in it's current implementation cannot download files from feeds that require authentication.

## Rationale and alternatives
By using an existing control we maintain consistency throughout the IDE and can rely on the owner to fix any bugs with the control.

Due to concerns about performance we will not be downloading the full nupkg temporarily just to access the README.

## Prior Art
The IMarkdownPreview is currently being used when creating a new pull request inside of Visual Studio.
![Alt text](../../meta/resources/READMEPMUI/PullRequestExperience.png) 

nuget.org currently renders the README and our users will expect them to look the same.
Ex. https://www.nuget.org/packages/Newtonsoft.Json#README-body-tab

## Unresolved Questions
1. Will we show README in the updates tab?
    - Yes
1. What do we show if the package has a README.txt instead of md?
    - The README is spec is written as only accepting MD.
    So we will use the nuspec to determine where the README is and treat it as md, even if the file is actually txt.
1. What do we show if there is no README defined?
    - Show the README tab with a message saying there is no README for the selected package/version.
1. Where do we get the README from when it's not on the disk?
    - Will document two ways for servers to provide the readme.
    Updating the package metadata to include a link to the readme, and a direct download similar to the way nuspec and nupkg are implemented.
1. Where are the README files saved in a package?
    - The nuspec defines where to locate the README.
1. What do we want the UX to be when an exception or error occurs while reading a README file?
    - Inform the users that an exception occurred. If time permits allow for a retry.
1. Do we want the README to update whenever a new version is selected for the current package?
   - Yes.

## Future Possibilities
Investigate ways for encouraging package owners to publish READMEs through this experience. 

Implement the option for users to opt out of rendering all images from README, similar to outlook with external images.

Update `INuGetPackageFileService` to allow for authenticated feeds.

Add caching to avoid downloading the README multiple times for the same URL.