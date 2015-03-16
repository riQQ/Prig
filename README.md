# Prig
## SYNOPSIS
Prig is a lightweight framework for test indirections in .NET Framework.



## DESCRIPTION
Prig(PRototyping jIG) is a framework that generates a [Test Double](http://martinfowler.com/bliki/TestDouble.html) like [Microsoft Fakes](http://msdn.microsoft.com/en-us/library/hh549175.aspx)/[Typemock Isolator](http://www.typemock.com/isolator-product-page)/[Telerik JustMock](http://www.telerik.com/products/mocking.aspx) based on Unmanaged Profiler APIs.
This framework enables that any methods are replaced with mocks. For example, a static property, a private method, a non-virtual member and so on.



## STATUS
As of Mar 16, 2015, Developing V2.0.0(Since this version, Prig is going to support Visual Studio integrated environment. More details can be found [here](https://github.com/urasandesu/Prig.V2Docs/blob/master/README.md)).  
As of Dec 31, 2014, Released V1.1.0.



## QUICK TOUR
Let's say you want to test the following code: 
```cs
using System;

namespace ConsoleApplication
{
    public static class LifeInfo
    {
        public static bool IsNowLunchBreak()
        {
            var now = DateTime.Now;
            return 12 <= now.Hour && now.Hour < 13;
        }
    }
}
```
You probably can't test this code, because `DateTime.Now` returns the value that depends on an external environment. To make be testable, you should replace `DateTime.Now` to the Test Double that returns the fake information. If you use Prig, it will enable you to generate a Test Double by the following steps without any editing the product code:


### Step 1: Install From NuGet
Run Visual Studio 2013(Express for Windows Desktop or more) as Administrator, add test project(e.g. `ConsoleApplicationTest`) and run the following command in the Package Manager Console: 
```powershell
PM> Install-Package Prig
```

**NOTE:** Installation will mostly go well. However, it doesn't go well if performing just after installing Visual Studio. [See also this issue's comment](https://github.com/urasandesu/Prig/issues/21#issuecomment-58741311).


### Step 2: Add Stub Settings
Run the following command in the Package Manager Console: 
```powershell
PM> Add-PrigAssembly -Assembly "mscorlib, Version=4.0.0.0"
```
The command means to create the indirection stub settings for the test. The reason to specify `mscorlib` is that `DateTime.Now` belongs `mscorlib`. After the command is invoked, you will get the confirmation message that the project has been modified externally, so reload the project.


### Step 3: Modify Stub Settings
You can find the setting file `<assembly name>.<runtime version>.v<assembly version>.prig` in the project(in this case, it is `mscorlib.v4.0.30319.v4.0.0.0.prig`). Modify the setting in accordance with the comment, then build all projects: 
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  
  <configSections>
    <section name="prig" type="Urasandesu.Prig.Framework.PilotStubberConfiguration.PrigSection, Urasandesu.Prig.Framework" />
  </configSections>

  <!-- 
      The content of tag 'add' is generated by the command 'Get-IndirectionStubSetting'.
      Specifically, you can generate it by the following PowerShell script in the Package Manager Console: 
      
      ========================== EXAMPLE 1 ==========================
      PM> $methods = Find-IndirectionTarget datetime get_Now
      PM> $methods
      
      Method
      ======
      System.DateTime get_Now()
      
      
      PM> $methods[0] | Get-IndirectionStubSetting | clip
      PM>
      
      Then, paste the clipboard content to between the tags 'stubs'.
  -->
  <prig>

    <stubs>
      <add name="NowGet" alias="NowGet">
        <RuntimeMethodInfo xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns:x="http://www.w3.org/2001/XMLSchema" z:Id="1" z:FactoryType="MemberInfoSerializationHolder" z:Type="System.Reflection.MemberInfoSerializationHolder" z:Assembly="0" xmlns:z="http://schemas.microsoft.com/2003/10/Serialization/" xmlns="http://schemas.datacontract.org/2004/07/System.Reflection">
          <Name z:Id="2" z:Type="System.String" z:Assembly="0" xmlns="">get_Now</Name>
          <AssemblyName z:Id="3" z:Type="System.String" z:Assembly="0" xmlns="">mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089</AssemblyName>
          <ClassName z:Id="4" z:Type="System.String" z:Assembly="0" xmlns="">System.DateTime</ClassName>
          <Signature z:Id="5" z:Type="System.String" z:Assembly="0" xmlns="">System.DateTime get_Now()</Signature>
          <Signature2 z:Id="6" z:Type="System.String" z:Assembly="0" xmlns="">System.DateTime get_Now()</Signature2>
          <MemberType z:Id="7" z:Type="System.Int32" z:Assembly="0" xmlns="">8</MemberType>
          <GenericArguments i:nil="true" xmlns="" />
        </RuntimeMethodInfo>
      </add>
    </stubs>
    
  </prig>

</configuration>
```


### Step 4: Make Tests
In the test code, it becomes testable through the use of the stub and the replacement to Test Double that returns the fake information: 
```cs
using NUnit.Framework;
using ConsoleApplication;
using System;
using System.Prig;
using Urasandesu.Prig.Framework;

namespace ConsoleApplicationTest
{
    [TestFixture]
    public class LifeInfoTest
    {
        [Test]
        public void IsNowLunchBreak_should_return_false_when_11_oclock()
        {
            using (new IndirectionsContext())
            {
                // Arrange
                PDateTime.NowGet().Body = () => new DateTime(2013, 12, 13, 11, 00, 00);

                // Act
                var result = LifeInfo.IsNowLunchBreak();

                // Assert
                Assert.IsFalse(result);
            }
        }

        [Test]
        public void IsNowLunchBreak_should_return_true_when_12_oclock()
        {
            using (new IndirectionsContext())
            {
                // Arrange
                PDateTime.NowGet().Body = () => new DateTime(2013, 12, 13, 12, 00, 00);

                // Act
                var result = LifeInfo.IsNowLunchBreak();

                // Assert
                Assert.IsTrue(result);
            }
        }

        [Test]
        public void IsNowLunchBreak_should_return_false_when_13_oclock()
        {
            using (new IndirectionsContext())
            {
                // Arrange
                PDateTime.NowGet().Body = () => new DateTime(2013, 12, 13, 13, 00, 00);

                // Act
                var result = LifeInfo.IsNowLunchBreak();

                // Assert
                Assert.IsFalse(result);
            }
        }
    }
}
```


### Step 5: Run Tests
In fact, to enable any profiler based mocking tool, you have to set the environment variables. Therefore, such libraries - Microsoft Fakes/Typemock Isolator/Telerik JustMock provide small runner to satisfy the requisition, also it is true at Prig. Use `prig.exe` and run the test as follows(continue in the Package Manager Console): 
```powershell
PM> cd <Your Test Project's Output Directory(e.g. cd .\ConsoleApplicationTest\bin\Debug)>
PM> prig run -process "C:\Program Files (x86)\NUnit 2.6.3\bin\nunit-console.exe" -arguments "ConsoleApplicationTest.dll /domain=None /framework=v4.0"
NUnit-Console version 2.6.3.13283
Copyright (C) 2002-2012 Charlie Poole.
Copyright (C) 2002-2004 James W. Newkirk, Michael C. Two, Alexei A. Vorontsov.
Copyright (C) 2000-2002 Philip Craig.
All Rights Reserved.

Runtime Environment - 
   OS Version: Microsoft Windows NT 6.2.9200.0
  CLR Version: 2.0.50727.8000 ( Net 3.5 )

ProcessModel: Default    DomainUsage: None
Execution Runtime: v4.0
...
Tests run: 3, Errors: 0, Failures: 0, Inconclusive: 0, Time: 0.0934818542535837 seconds
  Not run: 0, Invalid: 0, Ignored: 0, Skipped: 0

PM> 

```


### Final Step: Refactoring and Get Trig Back!
If tests have been created, you can refactor illimitably! For example, you probably can find the result of refactoring as follows: 
```cs
using System;

namespace ConsoleApplication
{
    public static class LifeInfo
    {
        public static bool IsNowLunchBreak()
        {
            // 1. Add overload to isolate from external environment then call it from original method.
            return IsNowLunchBreak(DateTime.Now);
        }

        public static bool IsNowLunchBreak(DateTime now)
        {
            // 2. Also, I think the expression '12 <= now.Hour && now.Hour < 13' is too complex.
            //    Better this way, isn't it?
            return now.Hour == 12;
        }
        // 3. After refactoring, no longer need to use Prig, because you can test this overload.
    }
}
```
As just described, Prig helps the code that depends on an untestable library gets trig back. I guarantee you will enjoy your development again!!

For more information, see also [Prig's wiki](https://github.com/urasandesu/Prig/wiki).




# INSTALLATION FROM SOURCE CODE
## DEPENDENCY
To build this project needs the following dependencies: 
* [Visual Studio 2013(more than Professional Edition because it uses ATL)](http://www.visualstudio.com/)
* [Boost 1.55.0](http://www.boost.org/)  
Extract to C:\boost_1_55_0, and will build with the following options(x86 and x64 libs are required):
```dos
CMD boost_1_55_0>cd
C:\boost_1_55_0

CMD boost_1_55_0>bootstrap.bat
Building Boost.Build engine

Bootstrapping is done. To build, run:

    .\b2

To adjust configuration, edit 'project-config.jam'.
Further information:
...

CMD boost_1_55_0>.\b2 link=static threading=multi variant=debug,release runtime-link=shared,static -j 4

Building the Boost C++ Libraries.

Performing configuration checks
...

CMD boost_1_55_0>.\b2 link=static threading=multi variant=debug,release runtime-link=shared,static -j 4 --stagedir=.\stage\x64 address-model=64

Building the Boost C++ Libraries.

Performing configuration checks
...
```
* [Google Test 1.6](https://code.google.com/p/googletest/)  
Extract to C:\gtest-1.6.0, and upgrade C:\gtest-1.6.0\msvc\gtest.sln to Visual Studio 2013. Choose the `Build` menu, and open `Configuration Manager...`. On `Configuration Manager` dialog box, in the `Active Solution Platform` drop-down list, select the `<New...>` option. After the `New Solution Platform` dialog box is opened, in the `Type or select the new platform` drop-down list, select a 64-bit platform. Then build all(Debug/Release) configurations.
* [NUnit 2.6.3.13283](http://www.nunit.org/)  
Install using with the installer(NUnit-2.6.3.msi).
* [Modeling SDK for Microsoft Visual Studio 2013](http://www.microsoft.com/en-us/download/details.aspx?id=40754)  
Install using with the installer(VS_VmSdk.exe).
* [Microsoft Visual Studio 2013 SDK](http://www.microsoft.com/en-us/download/details.aspx?id=40758)  
Install using with the installer(vssdk_full.exe).
* [Chocolatey NuGet](https://chocolatey.org/)  
Install the instructions in accordance with the page.
* [NAnt](http://nant.sourceforge.net/)  
You can also install in accordance with [the help](http://nant.sourceforge.net/release/latest/help/introduction/installation.html), but the easiest way is using Chocolatey: `choco install nant`.



## BUILD
After preparing all dependencies, you can build this project in the following steps:

1. Run Visual Studio as Administrator, and open Prig.sln(This sln contains some ATL projects, so the build process will modify registry).
2. According to the version of the product to use, change the solution configuration and the solution platform and build.
3. The results are output to `$(SolutionDir)$(Configuration)\$(PlatformTarget)\`.



## REGISTRATION
Run Developer Command Prompt for VS2013 as Administrator, and register dlls that were output to `$(SolutionDir)$(Configuration)\$(PlatformTarget)\` to registry and GAC as follows(these are the examples for x86/.NET 3.5, but also another environments are in the same manner): 
```dos
CMD x86>cd
C:\Prig\Release\x86

CMD x86>regsvr32 /i Urasandesu.Prig.dll

CMD x86>cd "..\..\Release(.NET 3.5)\AnyCPU"

CMD AnyCPU>cd
C:\Prig\Release(.NET 3.5)\AnyCPU

CMD AnyCPU>gacutil /i Urasandesu.NAnonym.dll
Microsoft (R) .NET Global Assembly Cache Utility.  Version 4.0.30319.33440
Copyright (c) Microsoft Corporation.  All rights reserved.

Assembly successfully added to the cache

CMD AnyCPU>gacutil /i Urasandesu.Prig.Framework.dll
Microsoft (R) .NET Global Assembly Cache Utility.  Version 4.0.30319.33440
Copyright (c) Microsoft Corporation.  All rights reserved.

Assembly successfully added to the cache

CMD AnyCPU>
```



## UNREGISTRATION
Unregistration operation is similar in the registration. Run Developer Command Prompt for VS2013 as Administrator and execute the following commands: 
```dos
CMD x86>cd
C:\Prig\Release\x86

CMD x86>regsvr32 /u Urasandesu.Prig.dll

CMD x86>cd "..\..\Release(.NET 3.5)\AnyCPU"

CMD AnyCPU>cd
C:\Prig\Release(.NET 3.5)\AnyCPU

CMD AnyCPU>gacutil /u "Urasandesu.Prig.Framework, Version=0.1.0.0, Culture=neutral, PublicKeyToken=acabb3ef0ebf69ce, processorArchitecture=MSIL"
Microsoft (R) .NET Global Assembly Cache Utility.  Version 4.0.30319.33440
Copyright (c) Microsoft Corporation.  All rights reserved.


Assembly: Urasandesu.Prig.Framework, Version=0.1.0.0, Culture=neutral, PublicKeyToken=acabb3ef0ebf69ce, processorArchitecture=MSIL
Uninstalled: Urasandesu.Prig.Framework, Version=0.1.0.0, Culture=neutral, PublicKeyToken=acabb3ef0ebf69ce, processorArchitecture=MSIL
Number of items uninstalled = 1
Number of failures = 0

CMD AnyCPU>gacutil /u "Urasandesu.NAnonym, Version=0.2.0.0, Culture=neutral, PublicKeyToken=ce9e95b04334d5fb, processorArchitecture=MSIL"
Microsoft (R) .NET Global Assembly Cache Utility.  Version 4.0.30319.33440
Copyright (c) Microsoft Corporation.  All rights reserved.


Assembly: Urasandesu.NAnonym, Version=0.2.0.0, Culture=neutral, PublicKeyToken=ce9e95b04334d5fb, processorArchitecture=MSIL
Uninstalled: Urasandesu.NAnonym, Version=0.2.0.0, Culture=neutral, PublicKeyToken=ce9e95b04334d5fb, processorArchitecture=MSIL
Number of items uninstalled = 1
Number of failures = 0

CMD AnyCPU>
```



