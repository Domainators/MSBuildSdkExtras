﻿# MSBuild.Sdk.Extras

## Summary

This package contains a few extra extensions to the SDK-style projects that are currently not available in `Microsoft.NET.Sdk` SDK. This feature is tracked in [dotnet/sdk#491](https://github.com/dotnet/sdk/issues/491) and many of the scenarios are on the roadmap for .NET 6.

The primary goal of this project is to enable multi-targeting without you having to enter in tons of properties within your `csproj`, `vbproj`, `fsproj`, thus keeping it nice and clean.

See the [blog post](https://claires.site/2017/01/04/multi-targeting-the-world-a-single-project-to-rule-them-all) for more information.

## Supported .NET Core SDK Versions

**Important:** 3.x of the Extras requires the .NET 5 SDK or later. The SDK can build previous targets, like `netcoreapp3.1`. The extras 2.x supports SDK 2.x and 3.x. 

## Advanced Scenarios

This package also enables advanced library scenarios, allowing you to create reference assemblies and per-RuntimeIdentifier targets. 

### Reference Assemblies
 
Reference Assemblies useful in a few scenarios. Please see my [two](https://claires.site/2018/07/09/create-and-pack-reference-assemblies-made-easy/) [blogs](https://claires.site/2018/07/03/create-and-pack-reference-assemblies/) for more details.

### Per-RuntimeIdentifier

In some cases involving native interop, it may be necessary to have different runtime versions. NuGet has supported this for a while if you use `PackageReference` by way of its `runtimes` folder in combination with a Reference Assembly. Creating and packing these were manual though.  

See [below](#rids) for creating these using the Extras easily.

## Package Name: `MSBuild.Sdk.Extras`

Stable: [![MSBuild.Sdk.Extras](https://img.shields.io/nuget/v/MSBuild.Sdk.Extras.svg)](https://nuget.org/packages/MSBuild.Sdk.Extras)

CI Feed: [![MSBuild.Sdk.Extras package in MSBuildSdkExtras feed in Azure Artifacts](https://feeds.dev.azure.com/clairernovotny/96789f1c-e804-4671-be78-d063a4eced9b/_apis/public/Packaging/Feeds/3773a966-220c-4410-a273-f6d772116a25/Packages/e25b4b0f-f40e-4fb7-8d25-7b266106b6b3/Badge)](https://dev.azure.com/clairernovotny/GitBuilds/_packaging?_a=package&feed=3773a966-220c-4410-a273-f6d772116a25&package=e25b4b0f-f40e-4fb7-8d25-7b266106b6b3&preferRelease=true) `https://pkgs.dev.azure.com/clairernovotny/GitBuilds/_packaging/MSBuildSdkExtras/nuget/v3/index.json`

### Getting started (VS 15.6+)

Visual Studio 2017 Update 6 (aka _v15.6_) includes support for SDK's resolved from NuGet, which is required for this to work. VS 2019 is recommended.

#### Using the SDK

1. Create a new project
    - .NET Core console app or .NET Standard class library.
    - With your existing SDK-style project.
    - With the templates in the repo's [TestProjects](/TestProjects) folder.

2. Replace `Microsoft.NET.Sdk` with `MSBuild.Sdk.Extras` to the project's top-level `Sdk` attribute.

3. You have to tell MSBuild that the `Sdk` should resolve from NuGet by
    - Adding a `global.json` containing the Sdk name and version.
    - Appending a version info to the `Sdk` attribute value.

4. Then you can edit the `TargetFramework` to a different TFM, or you can rename `TargetFramework` to `TargetFrameworks` and specify multiple TFM's with a `;` separator.

The final project should look like this:

```xml
<Project Sdk="MSBuild.Sdk.Extras">
  <PropertyGroup>
    <TargetFrameworks>net46;uap10.0.19041;tizen8.0</TargetFrameworks>
  </PropertyGroup>
</Project>
```

The .NET 5 SDK is the latest and has more support for desktop workloads. It's strongly recommended to use that SDK, even to build older targets. If you are using MsBuild.Sdk.Extras version 2 or above, use the .NET Core 3.1 SDK at a minimum. You can still target previous versions of .NET Core. 

```json
{
  "msbuild-sdks": {
    "MSBuild.Sdk.Extras": "3.0.22"
  }
}
```

Above the `sdk` section indicates use the .NET Core 3 preview to build, the `msbuild-sdks` indicates the NuGet package to include.

Then, all of your project files, from that directory forward, uses the version from the `global.json` file.
This would be a preferred solution for all the projects in your solution.

Then again, you might want to override the version for just one project _OR_ if you have only one project in your solution (without adding `global.json`), you can do so like this:

```xml
<Project Sdk="MSBuild.Sdk.Extras/3.0.22">
  <PropertyGroup>
    <TargetFrameworks>net46;uap10.0.19041;tizen8.0</TargetFrameworks>
  </PropertyGroup>
</Project>
```

That's it. You do not need to specify the UWP or Tizen meta-packages as they'll be automatically included.
After that, you can use the `Restore`, `Build`, `Pack` targets to restore packages, build the project and create NuGet packages. E.g.: `msbuild /t:Pack ...`

#### Important to Note

- It will only work with an IDE that uses the desktop `msbuild` (i.e. Visual Studio) and the target Platform SDKs which are not cross platform.
- When using JetBrains Rider, you will need to point to your desktop MSBuild in your settings (Settings > Build, execution, deployment > Use MSBuild Version)
- When building from the CLI, you must use `MSBuild.exe`. `dotnet build` **will not work** for most project types. 
- It might work in Visual Studio Code, but you have to configure build tasks in `launch.json` to use desktop `msbuild` to build.
- You must install the tools of the platforms you intend to build. For Xamarin, that means the Xamarin Workload; for UWP install those tools as well.

More information on how SDK's are resolved can be found [here](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-project-sdk#how-project-sdks-are-resolved).

### <a id="rids"></a>Creating Per-RuntimeIdentifier packages

You'll need to perform a few simple steps:

1. Make sure to use `TargetFrameworks` instead of `TargetFramework`, even if you're only building a single target framework. I am piggy-backing off of its looping capabilities.
2. Set the `RuntimeIdentifiers` property to [valid RID's](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog) ([full list](https://github.com/dotnet/runtime/blob/master/src/libraries/pkg/Microsoft.NETCore.Platforms/runtime.json)), separated by a semi-colon (`<RuntimeIdentifiers>win;unix</RuntimeIdentifiers>`).
3. For the TFM's that you want want to build separately, set the `ExtrasBuildEachRuntimeIdentifier` property to `true`.

When you're done, you should be able to run build/pack and it'll produce a NuGet package.

Notes:
* You must use the `Sdk="MSBuild.Sdk.Extras"` method for this. Using `PackageReference` is unsupported for this scenario.
* While the Visual Studio context won't show each RID, it'll build for each.
* The Extras defines a preprocessor symbol for each RID for use (`win-x86` would be `WIN_X86` and `centos.7-x64` would be `CENTOS_7_X64`). Dots and dashes become underbars.
* The default path for per-RID output assemblies and symbols in NuGet package is `runtimes/<RuntimeIdentifier>/lib/<TargetFramework>`.
* `RuntimeIdentifiers` can be set per-`TargetFramework` using a condition on the property. This lets you have multiple TFM's, but only some of which have RID's.

#### Reference Assemblies
You will likely need to create reference assemblies to simplify development and consumption of your libraries with complex flavor (`TargetFramework` × `RuntimeIdentifier`) matrix.
Reference assemblies are packed into `ref/<TargetFramework>` folder. Please see my [two](https://claires.site/2018/07/09/create-and-pack-reference-assemblies-made-easy/) [blogs](https://claires.site/2018/07/03/create-and-pack-reference-assemblies/) articles for details.

#### Packing additional contents
If you need to add native assets into runtimes, the easiest way is to use:
```xml
<None Include="path/to/native.dll" PackagePath="runtimes/<rid>/native" Pack="true" />
```

#### Overriding content paths in output package
Minimal example to pack output assemblies and symbols to `tools` (instead of `runtimes`) subfolders.
```xml
<PropertyGroup>
  <ExtrasIncludeDefaultProjectBuildOutputInPackTarget>IncludeDefaultProjectBuildOutputInPack</ExtrasIncludeDefaultProjectBuildOutputInPackTarget>
</PropertyGroup>

<Target Name="IncludeDefaultProjectBuildOutputInPack">
  <ItemGroup>
    <None Include="@(RidSpecificOutput)" PackagePath="tools/%(TargetFramework)/%(Rid)" Pack="true" />
  </ItemGroup>
</Target>
```

For advanced options, see _ClasslibPack*_ SDK tests and _RIDs.targets_ file.


### Migrate from the old way (VS pre-15.6)

For those who are using in a `PackageReference` style, you can't do that with v2.0+ of this package. So update VS to 15.6+ and manually upgrade your projects as shown below:

1. The same as above, replace the Sdk attribute's value.
2. Remove the workaround import specified with the old way. The import property should be `MSBuildSdkExtrasTargets`.
3. Do a trial build and then compare your project with the templates in the repo's [TestProjects](/TestProjects) folder to troubleshoot any issues if you encounter them.
4. Please file a issue. 

Your project diff:

```diff
- <Project Sdk="Microsoft.NET.Sdk">
+ <Project Sdk="MSBuild.Sdk.Extras">
  <!-- OTHER PROPERTIES -->
  <PropertyGroup>
    <TargetFrameworks>net46;uap10.0.16299;tizen40</TargetFrameworks>
  </PropertyGroup>

  <ItemGroup>
-    <PackageReference Include="MSBuild.Sdk.Extras" Version="1.6.0" PrivateAssets="All"/>
  <!-- OTHER PACKAGES/INCLUDES -->
  </ItemGroup>

-  <Import Project="$(MSBuildSdkExtrasTargets)" Condition="Exists('$(MSBuildSdkExtrasTargets)')"/>
  <!-- OTHER IMPORTS -->
</Project>
```

```diff
- PackageReference style
+ SDK style
```

**Note**: The SDK-style project now works on Visual Studio for Mac.

## Release Notes

### 1.6.0

 - A few properties have been changed, and the help is provided as a warning to use the new property names.

   | Old Property                                    | New Property/Behaviour                                             |
   | ---                                             | ----                                                               
   | `SuppressWarnIfOldSdkPack`                      | `ExtrasIgnoreOldSdkWarning`                                              |   
   | `ExtrasImplicitPlatformPackageDisabled`         | `DisableImplicitFrameworkReferences` + `TargetFramework` condition |
   | `EmbeddedResourceGeneratorVisibilityIsInternal` | opposite of `ExtrasEmbeddedResourceGeneratedCodeIsPublic`                |

 - Support for WPF and Windows Forms requires an opt-in property to enable:
Set `ExtrasEnableWpfProjectSetup`/`ExtrasEnableWinFormsProjectSetup` to `true` to include required references and default items. Note in .NET Core 3.0 these have been replaced by `UseWPF`/`UseWindowsForms`.

## Single or multi-targeting

Once this package is configured, you can now use any supported TFM in your `TargetFramework` or `TargetFrameworks` element. The supported TFM families are:

- `netstandard` (.NET Standard)
- `netcoreapp` (.NET Core App)
- `net` (.NET 5+ & .NET Framework)
- `net35-client`/`net40-client` (.NET Framework legacy Client profile)
- `wpa` (Windows Phone App 8.1)
- `win` (Windows 8 / 8.1)
- `uap` (Windows 10 / UWP)
- `wp` (Windows Phone Silverlight, WP7+)
- `sl` (Silverlight 4+)
- `tizen` (Tizen 3+)
- `xamarin.android`
- `xamarin.ios`
- `xamarin.mac`
- `xamarin.watchos`
- `xamarin.tvos`
- `portableNN-`/`portable-` (legacy PCL profiles like `portable-net45+win8+wpa81+wp8`)

 For legacy PCL profiles, the order of the TFM's in the list does not matter but the profile must be an exact match to one of the known profiles. If it's not, you'll get a compile error saying it's unknown. You can see the full list of known profiles here: [Portable Library Profiles by Stephen Cleary](https://portablelibraryprofiles.stephencleary.com/).

 If you try to use a framework that you don't have tools installed for, you'll get an error as well saying to check the tools. In some cases this might mean installing an older version of Visual Studio IDE (_like 2015_) to ensure that the necessary targets are installed on the machine.


