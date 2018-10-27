# autogen. Template processing script

Autogen gets template file + json file with values and produces one or multuple files from single template. Can be used to generate lot of code instead of writing manually.

## Prerequisites:
* python installed

## How to install

Run as a root:
```
curl -s "https://raw.githubusercontent.com/rustamkulenov/autogen/master/install.sh" | sudo bash -s
```
or:
```
$ pip install Jinja2
$ wget https://raw.githubusercontent.com/rustamkulenov/autogen/master/autogen.py
```

## File formats and structure
Template file is jinja2 file format. The following template will generate C# code for 'command' and interfaces for 'Listener' and 'Publisher':
```
{# Run 'python ./test.j2' to get file generated #}
/*
/* Autogenerated from {{ templateFileName }}
*/

using System;

{% for cmd in cmds %}
/// {{ cmd.comment }} command
public struct {{ cmd.name }}Cmd
{
{% for param in cmd.params %}
    /// {{ param.comment }}
    public {{ param.type }} {{ param.name}} { get; set; }
{% endfor %}
}
{% endfor %}

/// {{ name }}CmdListener
public interface I{{ name }}CmdListener
{
{% for cmd in cmds %}
    event EventHandler<{{ cmd.name }}Cmd> On{{ cmd.name }};
{% endfor %}    
}

/// {{ name }}CmdPublisher
public interface I{{ name }}CmdPublisher
{
{% for cmd in cmds %}
    /// {{ cmd.comment }}
    void {{ cmd.name }}({{ cmd.name }}Cmd cmd);
{% endfor %}        
}
```

Template file uses data from current context to render output.

Context file is json file (array of objects representing individual contexts). The folowing example of context file represents single channel 'OrderManager' with 2 methods (CloseAll and NewOrder):
```
[
    {
        "fileName": "OrderManager.cs",
        "name": "OrderManager",
        "cmds": [
            {
                "name": "CloseAll",
                "comment": "Close all orders",
                "params": [
                    {
                        "name": "AccountId",
                        "type": "long"
                    },
                    {
                        "name": "ExchangeId",
                        "type": "long"
                    }
                ]
            },
            {
                "name": "NewOrder",
                "comment": "Send new orders request",
                "params": [
                    {
                        "name": "AccountId",
                        "type": "long"
                    },
                    {
                        "name": "Symbol",
                        "type": "string"
                    },
                    {
                        "name": "Vol",
                        "type": "decimal"
                    }
                ]
            }
        ]
    }
]
```

Autogen iterates through contexts within context file and renders output files from template. Output file will have a name specified in 'fileName' key of a context and be placed near template file by default.

After executing
```
$ ./autogen ./test.j2
./test.j2 + ./test.j2.json => ./OrderManager.cs
```
Template and context files above will produce the following C# file: 
```
/*
/* Autogenerated from ./test.j2
*/
using System;

/// Close all orders command
public struct CloseAllCmd
{
    /// 
    public long AccountId { get; set; }
    /// 
    public long ExchangeId { get; set; }
}

/// Send new orders request command
public struct NewOrderCmd
{
    /// 
    public long AccountId { get; set; }
    /// 
    public string Symbol { get; set; }
    /// 
    public decimal Vol { get; set; }
}

/// OrderManagerCmdListener
public interface IOrderManagerCmdListener
{
    event EventHandler<CloseAllCmd> OnCloseAll;

    event EventHandler<NewOrderCmd> OnNewOrder;
}

/// OrderManagerCmdPublisher
public interface IOrderManagerCmdPublisher
{
    /// Close all orders
    void CloseAll(CloseAllCmd cmd);

    /// Send new orders request
    void NewOrder(NewOrderCmd cmd);       
}
```

## How to run
```
$ ./autogen.py ./<YOUR_TEMPLATE>.j2
```

## msbuild integration (dotnet core)
Automatic C# files generation can be performed during project build. Just please add <Target> and <ItemGroup> into your project file as shown below:
```
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <RootNamespace>cs_templates</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
      <J2 Include=".\**\*.j2"/>
  </ItemGroup>

  <Target Name="GenerateFilesTarget" AfterTargets="BeforeBuild" Condition="'@(J2)' != ''">
      <ItemGroup>
            <_FilesToProcess Include="@(J2)"/>
      </ItemGroup>

      <Exec Command="python autogen.py %(_FilesToProcess.Identity)" />
  </Target>


</Project>
```

<ItemGroup> will include all *.j2 files into build process, and <Target> will call autogen.py for each template file.
