---
title: Logging - How Automations Communicate
author: Keith Marshall
date: 2025-11-10
categories: [tutorial]
tags: [philosophy, tooling]
excerpt: A practical guide to building a reusable logging function in PowerShell — with clarity and intent.
layout: post
---

## Why logging matters
Logging acts as a communication layer between your code's execution and you, which creates an audit trail of observable data related to the execution. This hilights two key advantages of logging:
 - Make invisible execution observable.
 - Provide an audit trail for verifying success, or diagnosing failure.

## What should you log?
Logs should be clean, concise, meaningful, and most of all, written to be readable by the consumer.  
I use the term consumer intentionally. Logs aren't just for you; they can be consumed teammates, service technicians, monitoring services, dashboards, alerting systems, or other authomations... so format them with your target audience in mind.

## How to start logging
PowerShell includes built-in cmdlets that support interactive logging. However, since this blog focuses on automation, I’ll be exploring how to build non-interactive logging that works reliably in unattended scripts.

### Essentials
A log is essentially a formatted message that contains data for you about an automation's execution.

An example of generating a unicode log entry might look like this:

```powershell
$message = 'Hello, world'
$time    = Get-Date -Format yyyy-MM-ddTHH:mm:ss # ISO 8601 format
Write-Output -InputObject "$time :: $message"
```

I would say that this _is_ a log entry, but there are several areas of improvement.
 - Extract logic into a function.
 - Declare explicit data types to add calrity and error resistance.
 - Include metadata, like error levels or codes
 - Add persistence for monitoring or later analysis

### Proxy-functions and wrappers
Sometimes we need to expand or restrict a cmdlet’s behavior which can be done by "wrapping" the code in a function. We will do this with Write-Output to focus it away from general output into specialized log output. I will talk at length about wrappers and function development in a future post. For now let's break down our log develpment into stages.

- Extract the method into a function
- Tighten up the implemenation with data typing
- Add functionlity to meet expectation

#### First: Extract the Method into a Function
Declare a function name and signature and extract your method into it. Bonus points for shoe-horning 
[Microsoft PowerShell’s Approved Verbs](https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.5) into your design.

```powershell
function Write-Log ([string] $Message) {
    $time = Get-Date -Format yyyy-MM-ddTHH:mm:ss # ISO 8601 format
    Write-Output -InputObject "$Time :: $Message"
}
```
now we can call our wrapper with:
```Write-Log -Message "Hello, world"```

This adds a few advantages, the least of which is that we don't have to build $time for every log event, and secondly, we can extend Write-Log() with addtional functionality as we develop new log requirements.

#### Second: Be Explicit About Data Types
It's easy to let PowerShell infer variable data types, which is often fine, but being explicit about data types applies a level of readability and error resilience to your code in cases where assumptions about them don't hold up. Additionally, your code will have better tooling support for things like IntelliSense and static analysis, and make your code easier to test, review, and extend in the future.

In our Write-Log function it might be assumed that $time is of type DateTime instead of type String for example.

```powershell
function Write-Log ([string] $Message) {
    [string] $time = Get-Date -Format yyyy-MM-ddTHH:mm:ss # ISO 8601 format
    Write-Output -InputObject "$time :: $Message"
}
```

#### Third: Add More Value with a Log Level
A common extension of a log entry is a log level which allows the separation of logs into types for more granular analysis. In our example we are going to add four standard log levels:
- INFO for routine messages
- WARN for non-breaking errors
- ERROR for breaking errors
- DEBUG for detailed diagnositc information

To support log levels, we’ll convert our _basic_ function into an _advanced_ one by using a param() block. This lets us define acceptable values with ValidateSet, and set a default level of INFO to avoid breaking existing calls.”

```powershell
function Write-Log {
    param (
        [ValidateSet("INFO","WARN","ERROR","DEBUG")]
        [string] $LogLevel = INFO,

        [string] $Message
    )
    
    [string] $time = Get-Date -Format yyyy-MM-ddTHH:mm:ss # ISO 8601 format

    Write-Output -InputObject "$time :: [$LogLevel] :: $Message"
}
```
#### Fourth: Capturing the Logs
In this final step we'll capturing the logs in a local file. This raises a few questions like, what if the file does not exist? is Write-Output the right cmdlet to use for writing to a file, or should I consider Set/Add-Content?

There isn't a right answer, but I will stick with Write-Output since it is slightly more in alignment with my automation philosophies.

```powershell
function Write-Log {
    param (
        [ValidateSet("INFO","WARN","ERROR","DEBUG")]
        [string] $LogLevel = 'INFO',

        [string] $Message,
        [string] $Path
    )
    
    [string] $time = Get-Date -Format yyyy-MM-ddTHH:mm:ss # ISO 8601 format

    Write-Output -InputObject "$time :: [$LogLevel] :: $Message" | Out-File -Path $path -Encoding UTF8 -Append
}
```

### Extending the Idea Further
Logging is not a one-size-fits-all solution. For generating a local, human readable log with logging levels, this will suit most people, but if you need to build structured logs for consumption by a remote logging server, you may want to extend this function to include some data conversions like JSON, and at some point you may even consider building a logging class that contains a .ToJSON() method, but that is a topic for another day.

```powershell
function Write-Log {
    param (
        [ValidateSet("INFO","WARN","ERROR","DEBUG")]
        [string] $LogLevel = 'INFO',

        [string] $Message,
        [string] $Path,
        [switch] $ToJson
    )
    
    [string] $time = Get-Date -Format yyyy-MM-ddTHH:mm:ss # ISO 8601 format

    if ($ToJson) {
        $logObject = [PSCustomObject]@{
            Time     = $time
            LogLevel = $LogLevel
            Message  = $Message
        }
        $json = $logObject | ConvertTo-Json -Depth 3
        $json | Out-File -Path $Path -Encoding UTF8 -Append
        return
    }

    Write-Output -InputObject "$time :: [$LogLevel] :: $Message" | Out-File -Path $path -Encoding UTF8 -Append
}
```

## Final thoughts
Making logging a design habit is a key step toward building automation that’s not just functional, but reliable, readable, and built to last.