---
layout: post
title: Watching Problematic Services
subtitle: a stop-gap solution for stopping services
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [script, solution, services]
comments: True
---

## Background

Recently our team has been encountering an issue with a handful of custom services which tend to stop unexpectedly throughout the course of the day. While our monitoring tool does catch these, the polling interval of the tool can decrease our responsiveness. Additionally, it can become a hassle for off-shift personnel to continuously monitor and start services when there are other tasks which need to be done.

>**Note: This is not intended as a permanent solution, this is a stop-gap which should be use judiciously while the root cause of the stopping services are investigated**

## Getting The Services

Pulling the services from the remote computer or computers can be accomplished simply by passing a computer name or an array of computers to the **Get-Service** command. First we assign the computer name to a variable to make modification later easier.

```powershell
$computer = "computername"
$computer = @("computer1","computer2")
```

Services can sometimes be in a degraded, stopping, or other state which is not 'Stopped' but is not running. We want to target any which are not Running.

```powershell
$StoppedServices = Get-Service -ComputerName $Computer | Where-Object {($_.StartType -eq "Automatic") -and ($_.Status -ne "Running")}
$StoppedServices # Output Services
```

For a simple situation of *"We want all automatic services on these computers running"* the above may be sufficient. However, there are likely some services that either we do not want to start that intermittently run automatically or are manual services which we are concerned about. In this instance we want to specifically locate services based on their display name.

Piping the output of **Get-Service** to **Where-Object** allows us to run comparisons against the return objects and only returns **objects** **where** our conditions return True.

I decided to use the **-match** operator as it allows the flexibility of performing partial matches on names (*e.g "FooBar" -match "Foo" returns True*). This fit our specific use case considering all the Services we were concerned with contained a particular phrase. In the event one only wants to match on an exact name, using **-eg** is better suited (*e.g. "FooBar" -eg "Foo" returns False, but "FooBar" -eq "FooBar" returns True*)

```powershell
# Specify the Partial Service Name
$Service = "Foo"

# Watch Specific Services
$WatchServices = Get-Service -ComputerName $Computer | Where-Object {$_.Displayname -match $Service}

# Compare Stopped to specific watched services and output to see the overlap
$StoppedServices | Where-Object {$WatchServices.ServiceName -contains $_.ServiceName}
```

Of course simply knowing which of the Services we are watching are not running does not help our operational metrics. We actually want to attempt to start the services.

## Starting The Services

The command **Start-Service** is what we can leverage in this instance to attempt to start up the service on the remote computers. However, with this we may run into a problem where execution takes a long time if there are a large number of stopped services. If you are using PowerShell Version 5 or newer, you can add the **-NoWait** parameter. The full command will look something like the below

```powershell
Start-Service -name <ServiceName> -ComputerName <Computer> -NoWait
```

Unfortunately, the environment I was working with at the time was using an older version of PowerShell without this capability. Fortunately, this same parallelism can be achieved through **Invoke-Command** and using the **-asjob** parameter to make the command run against a certain computer as a job in the background. The **-ScriptBlock**, where we pass the command to run, cannot directly accept our local variable. Instead this is passed in an **-ArgumentList** which creates an array of any variables we may want to pass to the remote computer. The specific variable is then retrieved by specifying its place in the **$args** array.

```powershell
Invoke-Command -ComputerName <ComputerName> -AsJob -ScriptBlock { Start-Service -name $args[0]} -ArgumentList <ServiceToStart>
```

## Putting it All Together

Finally, we put this together with the PowerShell written previously that finds stopped services we want to watch and invoke the command to start the service for each that is not running. The logic performs as follows:

>Take all the Services we are watching --> Check if Any Are stopped --> Start a background job to start any stopped services --> Wait for all the background jobs to complete

All the PowerShell to accomplish this is put together below:

```powershell
# Specify the Computers to Check
$computer = @("computer1","computer2")

# Specify the Partial Service Name
$Service = "Foo"

# Watch Specific Services
$WatchServices = Get-Service -ComputerName $Computer | Where-Object {$_.Displayname -match $Service}

# Attempt to Start up Watched services that are in a 'stopped' state as a remotely invoked job
$WatchServices |
     Where-Object {$_.Status -ne "Running"} |
         ForEach-Object {
            Invoke-Command -ComputerName $_.MachineName -AsJob -ScriptBlock {Start-Service -name $args[0] } -ArgumentList $_.ServiceName | Out-Null
        }

Get-Job #See the jobs we started
Get-Job | Wait-Job #Wait for jobs to complete

$Output = Get-Job | Receive-Job # Get Job output so we don't lose it
$Output # Out Job output

# Check again the status of our watch services
Get-Service -ComputerName $Computer | Where-Object {$_.DisplayName -Match $Service}
```
