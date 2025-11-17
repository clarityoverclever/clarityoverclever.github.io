---
title: "Organizing PowerShell Projects: A Practical Framework"
author: "Keith Marshall"
date: 2025-11-15
categories: [tutorial]
tags: [patterns]
excerpt: Learn how to structure PowerShell projects with a practical framework that keeps your code organized and reusable
layout: post
---

## Organization Matters
As PowerShell projects grow beyond a few scripts, their complexity can become overwhelming, derailing progress and leaving you feeling disorganized. In this post, I'll share an organizational plan that helps keep projects extensible, maintainable, and explainable.

At the core, the approach is simple: break tasks into small, reusable functions that don’t rely on shared state.

## Start with Structure
I've been using the following structure in my code for years and find it to be a good balance of simplicity and functionality. I prefer to lowercase my directory names to aid in platform cross compatibility.

### Diagram
```
├── config  
│   ├── config.ps1  
├── lib  
│   ├── function1.ps1  
│   ├── function2.ps1  
│   ├── Write-Log.ps1  
├── log  
│   ├── file.log  
├── public  
│   ├── launcher.ps1
```
### config
The **config** directory is where I store the variable parts of the application that define how it will run. This includes the actual configuration and also files like credential objects that can be used for authentication.  

### lib
The **lib** directory is the heart of the project, and stores the logic which the application uses. Each function lives in its own file, abstracting logic out of the main control script. This leaves isolated code blocks that are easy to reuse, extend, and test.

### log
The **log** directory provides a dedicated space for diagnostic output. Centralizing logs ensures that troubleshooting is predictable. I have more to say about logging [here](https://clarityoverclever.github.io/posts/logging-how-automations-communicate/)

### public
The **public** directory contains files that users are meant to interact with directly, contrasted with **lib** where code files used by the application are stored. Most notably, the application launcher will be located here.

### other
There are scenarios where other directory might be appropriate, some examples include:
 - **form** : to contain front end files like XAML pages for Windows WPF forms
 - **data** : to contain data used by the application like key stores or databases
 - **out**  : to contain exported data

## Pathing Patterns
Once you start building your application, you’ll find that pathing needs to be treated as a relative construct to the launcher. There are some simple patterns you can use to tie the logic together.

It’s generally safe to set paths as regular variables, but because paths don’t change during execution, you may consider making them immutable. This practice enforces stability by preventing accidental reassignment and helps catch errors early.

Note: I prefer to use `snake_case` for standard path variables and `ALL_CAP_SNAKE_CASE` for immutable path variables.

```powershell
# mutable path variables for flexibility
[string] $root_path    = Split-Path -Path $PSScriptRoot -Parent
[string] $config_path  = Join-Path -Path $root_path -ChildPath 'config'
[string] $lib_path     = Join-Path -Path $root_path -ChildPath 'lib'
[string] $log_path     = Join-Path -Path $root_path -ChildPath 'log'

# immutable path variables for stability
Set-Variable -Name ROOT_PATH   -Value (Split-Path -Path $PSScriptRoot -Parent) -Option ReadOnly
Set-Variable -Name CONFIG_PATH -Value (Join-Path -Path $ROOT_PATH -ChildPath 'config') -Option ReadOnly
Set-Variable -Name LIB_PATH    -Value (Join-Path -Path $ROOT_PATH -ChildPath 'lib')    -Option ReadOnly
Set-Variable -Name LOG_PATH    -Value (Join-Path -Path $ROOT_PATH -ChildPath 'log')    -Option ReadOnly
```
> **Immutable Variables:** are automatically discarded when the script ends; however, you may want to explicitly remove them with `Remove-Variable -Name VARIABLE -Force`.

## Wiring in the Function Library
Importing functions into the automation is done by _dot sourcing_ the function(s). This can be done explicitly, when you want granular control over what is loaded by your automation, or recursively when you want to simplify function loading.

```powershell
# explicitly dot source a function
. (Join-Path -Path $LIB_PATH -ChildPath 'Write-Log.ps1')

# recursively dot source the lib directory
foreach ($function in Get-ChildItem -Path $LIB_PATH -Filter *.ps1) {
    . $function.FullName
}
```
> **Notes on Recursive Loading:**  
> When recursively dot sourcing, be mindful of what lives in your `lib` directory. The loop will load *every* `.ps1` file it finds, including test scripts, experimental helpers, or files you didn’t intend to expose.  
> 
> To avoid surprises:  
> - Keep only production‑ready functions in `lib`.  
> - Move prototypes or scratch scripts into a separate directory (e.g., `sandbox`).  
> - Use clear naming conventions so you know at a glance what’s safe to load.  
{: .note }

## Pulling in the Config
As mentioned above, the configuration file holds the variable parameters that you can use to adjust the behavior of your automation. I prefer to store the configuration in a hashtable which, like other functions, is ingested by dot sourcing.

### Define the Config
This simple config illustrates a basic workflow pattern for defining and ingesting a configuration. I am using the variable $config by convention, you can name this however you like.

```powershell
# config/config.ps1
$config = @{
    app_name      = 'MyAutomation'
    environment   = 'dev'
    log_level     = 'info'
    api_endpoint  = 'https://api.example.com'
    credentials   = Get-Credential
}
```
> **Notes on Credentials:**  
> In the above example I have a call to Get-Credential which will prompt the user for a credential every time the automation runs. This is probably not an ideal scenario, and I will discuss at length ways to store credentials or elevate an automation's session, in a future post.
> 
> Considerations :  
> - Never store plaintext credentials in script files, use a secure storage mechanism instead.  
{: .note }

### Dot Source the Config and Accessing its Properties
```powershell
# public/launcher.ps1
. (Join-Path -Path $CONFIG_PATH -ChildPath 'config.ps1')

# access the config parameters in a quoted string
Write-Host "Launching $($config.app_name) in $($config.environment) mode..."

# access the config parameters standardly
$logFile = (Join-Path -Path $LOG_PATH -ChildPath (Get-Date -UFormat %A))
Write-Log -Path $logFile -Message 'Sweet little log message' -LogLevel $config.log_level

# switch behavior based on config variable
$env = ($config.environment).ToLower() 

if ($env -eq 'dev') {
    Write-Host "environment is: $env"
}
```
### A Multi-Environment Example:
You may need different configurations to represent behaviors across multiple environments. This can be done by sourcing dedicated config files for each environment, or by combining them into a single file using nested hashtables, like below, and adding a parameter to your launcher to dynamically select the environment you are loading at execution.

```powershell
# config/config.ps1
$configs = @{
    dev = @{
        app_name     = 'MyAutomation'
        environment  = 'dev'
        log_level    = 'debug'
        api_endpoint = 'https://api-dev.example.com'
    }
    prod = @{
        app_name     = 'MyAutomation'
        environment  = 'prod'
        log_level    = 'error'
        api_endpoint = 'https://api.example.com'
    }
}

# public/launcher.ps1
. (Join-Path -Path $CONFIG_PATH -ChildPath 'config.ps1')

param(
    [string]$env = 'prod'  # add a -env parameter to the launcher which defaults to prod but can be overridden
)

$config = $configs[$env]

Write-Host "Launching $($config.app_name) in $($config.environment) mode..."
```

## Wrapping Up
A little structure goes a long way. By separating configuration, logic, logs, and public entry points, you make your PowerShell projects easier to maintain, easier to share, and easier to grow. This is a foundation that can be added upon as your needs evolve.  

The key is consistency: when your future self (or a teammate) revisits the project, the organization will speak for itself.
