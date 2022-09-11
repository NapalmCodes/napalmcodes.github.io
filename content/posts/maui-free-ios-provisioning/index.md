+++
author = "Shawn Vause"
title = ".NET MAUI iOS Free Provisioning"
date = "2022-09-11"
tags = [
    "dotnet",
    "maui",
    "ios",
    "mobile"
]
draft = true
+++

Contrary to popular belief, it is possible to run an iOS application on your iPhone/iPad without paying the $99 USD fee per year to Apple for a developer program account. Through a process called free provisioning we can gain this freedom. It is a relatively simple task to accomplish and only involves a some clicking around Xcode. There is no jailbreaking involved and it is supported by Apple. It does have some limitations that we will get into, but for those learning mobile development in Apple's walled garden platform, it is a great way to get started without the added expense.

We will demonstrate how free provisioning gets us started running code on our iOS device, through the lens of the relatively new <a href="https://docs.microsoft.com/en-us/dotnet/maui/what-is-maui" title=".NET MAUI">.NET MAUI platform</a>. For those unaware, .NET MAUI is a solution that allows us to write a single UI using XAML or Blazor based controls that will run across iOS, Android, Mac and Windows. There are also capabilities to call into platform specific code as needs dictate, for instance to access sensors or platform specific APIs like those provided for in-app purchases. A simulator does exist for iOS on the Mac and Visual Studio provides some great tooling around remoting to your simulator from Windows environments. However, there is no substitute for running your software on physical hardware while developing. Frankly, it is a faster experience and a lot closer to how your customers will use your application.
{{<preview_ph>}}

### Requirements

- Visual Studio 2022 17.3.0 or above with the .NET MAUI workload installed
- A Mac to compile and plug your iPhone/iPad device into
- Xcode latest version installed on your Mac
- An Apple ID not connected to Apple Developer Program

### Xcode Steps

Begin by opening Xcode on your Mac and navigating via the menu bar to *Xcode > Preferences*. Under the *Accounts* tab, click the plus button to add your Apple ID account to Xcode using your user name and password.
<br/><br/>
<img src="xcode-add-acct.jpg" alt="Xcode Add Account Dialog" style="display: block; margin: 0 auto; length: 700px; width: 600px" />
