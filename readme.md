# <img src="/package_icon.png" height="30px"> PropertyChanged.Fody

[![NuGet Status](https://img.shields.io/nuget/v/PropertyChanged.Fody.svg)](https://www.nuget.org/packages/PropertyChanged.Fody/)

Injects code which raises the [`PropertyChanged` event](https://msdn.microsoft.com/en-us/library/system.componentmodel.inotifypropertychanged.propertychanged.aspx), into property setters of classes which implement [INotifyPropertyChanged](https://msdn.microsoft.com/en-us/library/system.componentmodel.inotifypropertychanged.aspx).


**See [Milestones](../../milestones?state=closed) for release notes.**

## This is an add-in for [Fody](https://github.com/Fody/Home/)

**It is expected that all developers using Fody [become a Patron on OpenCollective](https://opencollective.com/fody/contribute/patron-3059). [See Licensing/Patron FAQ](https://github.com/Fody/Home/blob/master/pages/licensing-patron-faq.md) for more information.**


## Usage

See also [Fody usage](https://github.com/Fody/Home/blob/master/pages/usage.md).


### NuGet installation

Install the [PropertyChanged.Fody NuGet package](https://nuget.org/packages/PropertyChanged.Fody/) and update the [Fody NuGet package](https://nuget.org/packages/Fody/):

```powershell
PM> Install-Package Fody
PM> Install-Package PropertyChanged.Fody
```

The `Install-Package Fody` is required since NuGet always defaults to the oldest, and most buggy, version of any dependency.


### Add to FodyWeavers.xml

Add `<PropertyChanged/>` to [FodyWeavers.xml](https://github.com/Fody/Home/blob/master/pages/usage.md#add-fodyweaversxml)

```xml
<Weavers>
  <PropertyChanged/>
</Weavers>
```


## Overview

**NOTE: All classes that implement `INotifyPropertyChanged` will have notification code injected into property setters.**

Before code:

```csharp
public class Person : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    
    public string GivenNames { get; set; }
    public string FamilyName { get; set; }
    public string FullName => $"{GivenNames} {FamilyName}";
}
```

What gets compiled:

```csharp
public class Person : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    string givenNames;
    public string GivenNames
    {
        get => givenNames;
        set
        {
            if (value != givenNames)
            {
                givenNames = value;
                OnPropertyChanged(InternalEventArgsCache.GivenNames);
                OnPropertyChanged(InternalEventArgsCache.FullName);
            }
        }
    }

    string familyName;
    public string FamilyName
    {
        get => familyName;
        set 
        {
            if (value != familyName)
            {
                familyName = value;
                OnPropertyChanged(InternalEventArgsCache.FamilyName);
                OnPropertyChanged(InternalEventArgsCache.FullName);
            }
        }
    }

    public string FullName => $"{GivenNames} {FamilyName}";

    protected void OnPropertyChanged(PropertyChangedEventArgs eventArgs)
    {
        PropertyChanged?.Invoke(this, eventArgs);
    }
}

internal static class InternalEventArgsCache
{
    internal static PropertyChangedEventArgs FamilyName = new PropertyChangedEventArgs("FamilyName");
    internal static PropertyChangedEventArgs FullName = new PropertyChangedEventArgs("FullName");
    internal static PropertyChangedEventArgs GivenNames = new PropertyChangedEventArgs("GivenNames");
}
```

(the actual injected type and method names are different)


## Code Generator

Starting with version 4 PropertyChanged.Fody ships with a C# code generator that can even more simplify your code by generating
the boilerplate of the basic `INotifyPropertyChanged` implementation for you directly as source code.

Simply mark a class implementing `INotifyPropertyChanged` or having the `[AddINotifyPropertyChangedInterface]` attribute as `partial` and the
generator will add the necessary event and event-invokers:

e.g. a class like this:

```c#
public partial class Class1 : INotifyPropertyChanged
{
    public int Property1 { get; set; }
    public int Property2 { get; set; }
}
```

will be complemented by the generator with this:

```c#
public partial class Class1
{
    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        OnPropertyChanged(new PropertyChangedEventArgs(propertyName));
    }
    protected virtual void OnPropertyChanged(PropertyChangedEventArgs eventArgs)
    {
        PropertyChanged?.Invoke(this, eventArgs);
    }
}
```

- Only classes are supported, no records.
- For nested classes, all containing classes must be partial, too.
- Code generators only work correctly in SDK-style projects

### Code Generator Configuration

You can configure the code generator via properties in your project file:

```xml
<PropertyGroup>
  <PropertyChangedAnalyzerConfiguration>
    <IsCodeGeneratorDisabled>false</IsCodeGeneratorDisabled>
    <EventInvokerName>OnPropertyChanged</EventInvokerName>
  </PropertyChangedAnalyzerConfiguration>
</PropertyGroup>
```

- *IsCodeGeneratorDisabled*: Set to `true` to switch off the code generator.
- *EventInvokerName*: Change the name of the event invoker method from `OnPropertyChanged` to your favorite name.


### Workaround for WPF projects targeting multiple frameworks:

WPF projects targeting multiple frameworks may fail during the compilation of the `*_wpftmp.csproj` with 

`... error CS0111: Type 'SomeType' already defines a member called 'OnPropertyChanged' with the same parameter types`

This can be fixed by adding this build target to your project:

```xml
<Target Name="RemoveDuplicateAnalyzers" BeforeTargets="CoreCompile">
  <!-- see https://github.com/dotnet/wpf/pull/6680 -->
  <RemoveDuplicates Inputs="@(Analyzer)">
    <Output
      TaskParameter="Filtered"
      ItemName="FilteredAnalyzer"/>
  </RemoveDuplicates>
  <ItemGroup>
    <Analyzer Remove="@(Analyzer)" />
    <Analyzer Include="@(FilteredAnalyzer)" />
  </ItemGroup>
</Target>
```

---


## Notes

- **Dependent properties** — In the above sample, the getter for `FullName` depends on the getters for `GivenName` and `FamilyName`. Therefore, when either `GivenName` or `FamilyName` is set, `PropertyChanged` is raised for `FullName` as well.   This behavior can be configured manually using the [`AlsoNotifyFor` attribute](https://github.com/Fody/PropertyChanged/wiki/Attributes#alsonotifyforattribute) on the source property, or the [`DependsOn` attribute](https://github.com/Fody/PropertyChanged/wiki/Attributes#dependsonattribute) on the target property).
- **Intercepting the notification call**
  - [**Global interception**](https://github.com/Fody/PropertyChanged/wiki/NotificationInterception)
  - **Class-level interception** — The `OnPropertyChanged` method will only be injected if there is no such existing method on the class; if there is such a method, then calls to that method will be injected into the setters — see [here](https://github.com/Fody/PropertyChanged/wiki/EventInvokerSelectionInjection).
  - **Property-level interception** — For a given property, if there is a method of the form `On<PropertyName>Changed`, then that method will be called — see [here](https://github.com/Fody/PropertyChanged/wiki/On_PropertyName_Changed).
- To get the [**before / after values**](https://github.com/Fody/PropertyChanged/wiki/BeforeAfter), use the following signature for `OnPropertyChanged` / `On<PropertyName>Changed`:

      public void OnPropertyChanged(string propertyName, object before, object after)
- To prevent a specific class from having the notification call injection, use the [`DoNotNotify` attribute](https://github.com/Fody/PropertyChanged/wiki/Attributes#donotnotifyattribute).
- To scope the rewriting only to specific classes, and not the whole Assembly, you can use the [`FilterType` attribute](https://github.com/Fody/PropertyChanged/blob/f4905b4f04335e393c8350cc5f06f02614241483/PropertyChanged.Fody/TypeNodeBuilder.cs#L18). This changes the general behavior from from opt-out to opt-in. Example: `[assembly: PropertyChanged.FilterType("My.Specific.OptIn.Namespace.")]`. The string is interpreted as a Regex, and you can use multiple filters. A class will be weaved, if *any* filter matches.
- The `INotifyPropertyChanged` interface can be automatically implemented for a specific class using the [`AddINotifyPropertyChangedInterfaceAttribute` attribute](https://github.com/Fody/PropertyChanged/wiki/Attributes#addinotifypropertychangedinterfaceattribute). **Raising an issue about "this attribute does not behave as expected" will result in a RTFM and the issue being closed.**
  - for partial methods this will be done via the code generator, so the implementation is available at compile time.
- Behavior is configured via [attributes](https://github.com/Fody/PropertyChanged/wiki/Attributes), or via [options in the `Weavers.xml` file](https://github.com/Fody/PropertyChanged/wiki/Options).

For more information, see the [wiki pages](https://github.com/Fody/PropertyChanged/wiki).


## Icon

Icon courtesy of [The Noun Project](https://thenounproject.com)
