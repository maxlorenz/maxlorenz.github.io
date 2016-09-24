---
layout: post
title:  "Qt Project Layout"
date:   2016-09-24 14:00:00 +0200
categories: c++ qt
---
In order to create an Qt project with tests, I stumbled upon a very helpful tutorial and repository [here](https://github.com/ComputationalPhysics/qtcreator-project-structure).

However, I was not fully satisfied. For one, the library was missing the `__declspec` attribute so the DLL would not work. Besides, I couldn't find a way to automate my builds (so you could use build servers or add some simple CI).

In this write-up, I will cover the following things:
- Splitting a Qt project into 3 sub-projects (A library, tests and a GUI)
- Using QtTest for TDD
- Automating builds with PS
- Using the MS C++ compiler

## Initial setup

[Here](https://github.com/maxlorenz/qt-project-structure) is what I came up with.

After installing QtCreator and Visual Studio, you simply create a new subdirs project. Then, you create your Library (_rightclick, New Subproject_), your tests (_Other Project, Qt Unit Test_) and your actual app (_Application, Qt Widgets Application_). I chose this structure to have the business logic in an interface agnostic library with separated tests, while the UI has it's own project.

After creating the projects, I added the library to the application's and test's project file. To add the `__declspec` attribute to the library, I created a simple `samplelibrary_global.h` header, as suggested in the [Qt wiki](https://wiki.qt.io/How_to_create_a_library_with_Qt_and_use_it_in_an_application). This enables DLL export of classes like this:

{% highlight c++ %}
#include "samplelibrary_global.h"

class SAMPLELIBRARYSHARED_EXPORT SampleLibrary
{

public:
    SampleLibrary();
    int add(int a, int b);
};
{% endhighlight %}

## Test

When I added the Qt Test plugin, I could see the test results with `alt+shift+t, alt+a` so I could do some TDD. QtTest has some quirks but works fine for my purposes.

## Build automation

The most difficult task was to automate the build. Qt has some windows deployment tools, but they didn't work the way I wanted them to so I wrote a little Powershell script.

Surprisingly, I couldn't find a lot of help on SO that was Windows specific. To build the project, Qt 

- runs Qmake
- gets the environment variables from `vcvarsall.bat`
- builds the program using `jom.exe`

For deployment, you still have to copy the Qt DLLs. I could replicate that in PS (Using MS C++ compiler, x64, release build):

{% highlight powershell %}
# Get VS and Qmake bin path
$VS_REG = "HKLM:\SOFTWARE\Wow6432Node\Microsoft\VisualStudio"
[xml]$qt_conf = Get-Content "$env:APPDATA\QtProject\qtcreator\qtversion.xml"

$VS = @("$VS_REG\14.0", "$VS_REG\12.0", "$VS_REG\10.0") `
| Where-Object { Test-Path $_ } `
| ForEach-Object { Get-ItemProperty $_ | Select-Object -ExpandProperty "ShellFolder" }

$Qmake = $qt_conf.qtcreator.data[0].valuemap.ChildNodes `
| Where-Object { $_.key -eq "QMakePath" } `
| Select-Object -ExpandProperty "#text" -First 1

# Load environment variables
& "$VS\VC\vcvarsall.bat" x64 "&set" `
| ForEach-Object { $k, $v = $_.split("="); [System.Environment]::SetEnvironmentVariable($k, $v, "Process") }

if (!(Test-Path dist)) { New-Item -ItemType Directory -Path dist }
Set-Location .\dist

# Build
& "$Qmake" ../SampleProject.pro -spec win32-msvc2015 CONFIG+=release
& C:\Qt\Tools\QtCreator\bin\jom.exe /S

# Copy libs + exes
$Qt_libs = [System.IO.Path]::GetDirectoryName($Qmake)
@("Qt5Core.dll", "Qt5Gui.dll", "Qt5Test.dll", "Qt5Widgets.dll") `
| ForEach-Object { Copy-Item "$Qt_libs\$_" -Force }

Get-ChildItem -Path ".\*\*" -Recurse -Include "*.exe", "*.dll" `
| ForEach-Object { Copy-Item $_ -Force }

# Show test results
& ".\tst_sampletest.exe"
Set-Location ..
{% endhighlight %}

[Link](https://github.com/maxlorenz/qt-project-structure) to the finished project