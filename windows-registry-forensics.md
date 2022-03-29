# Windows Registry Forensics

### Overview

What is the Windows Registry?

> From Microsoft: "The registry is a system-defined database in which applications and system components store and retrieve configuration data."

In other words, this is the location that Operating System and Installed programs can store their configurations and possibly the history of a program/user/so has done.

Why is this important for us in the world of Cyber Defenders/Digital Forensics matter to us?

There is a ton of information that can be extracted from the registry if you know where to look. some of the big items that we can find in the registry include:

* Startup operations
* Current and Past networks connected to
* Recent user activities
* Devices connected to the system
* Printers that have been used
* Services and their properties/configurations
* Network shares connected to
* Persistence of malicious programs

This is just a short list of the items that we may find of interest in hunting through the Windows registry, but how do we go about and look into these items? There are so many registry entries in a windows system.

Windows helps keep things organized by splitting things up in separate registry hives.

There are a handful of hives that all have very important uses to the way the Window Operating System works.

* HKEY\_CLASSES\_ROOT (HKCR)
* Contains information about registered applications, this is a combination of the current\_user and local\_machine classes subkeys where the current user information will override a matching local\_machine registry key.
* HKEY\_CURRENT\_USER (HKCU)
* Registry and settings of the current logged in user. Data here is pulled from the ntuser.dat file
* HKEY\_LOCAL\_MACHINE (HKLM)
* Stores settings that are specific to the local computer. These keys are loaded at boot located %systemroot%\system23\config folder
* HKEY\_USERS (HKU)
* Similar to HKEY\_CURRENT\_USER but rather than the current logged in user it tracks all the profiles currently stored on the computer.
* HKEY\_CURRENT\_CONFIG (HKCC)
* pointer to settings currently in use by the computer. Most of the items point to the HKLM hive. Changes made to HKLM\SYSTEM\CurrentControlSet\Hardware Profiles\Current\ typically mirror the HKCC and vice versa.

The registry is your typical database where you have keys and values.

* Keys are similar to folders in the filesystem. They are used as a way to organize the data in the registry
* Values are name/data pairs that are stored inside of a registry key. This of this like the documents inside of a folder.

The registry can contain several types of values in it, so it can be more than just numbers or words. Here is a quick chart to cover those various items:

| Type Name           | Meaning                                                             |
| ------------------- | ------------------------------------------------------------------- |
| REG\_NONE           | No type (the stored value, if any)                                  |
| REG\_SZ             | A string value                                                      |
| REG\_EXPAND\_SZ     | An "expandable" string value that can contain environment variables |
| REG\_BINARY         | Binary data                                                         |
| REG\_DWORD          | a 32-bit unsigned integer                                           |
| REG\_LINK           | A symbolic link to another registry key                             |
| REG\_MULTI\_SZ      | A multi-string value, which is an ordered list of non-empty strings |
| REG\_RESOURCE\_LIST | A resource list                                                     |
| REG\_QWORD          | A 64-bit integer                                                    |

### RegScout

I want to introduce a tool that I have developed in order to extract important information from windows computers. I went with a modularized set of PowerShell scripts that are prepared to be used in a framework and currently set to be shipped back to an Elasticsearch instance for analysis.

Here is a list of the modules and a brief description as to what they were built for:

| Module Name          | Description                                                                                                                                                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Computer Info        | List of several computer baseline items, processor info, system environment, RDP enabled, prefetch, paging, lastuser logged in.                                                                                                 |
| Downloaded Docs      | This will list all items that are asked for a prompt to open/save files (last 20 per extension)                                                                                                                                 |
| Enum                 | List of any device connected to a computer (USBs will be in this list)                                                                                                                                                          |
| File Extensions      | List of all Extensions and default apps assigned per user and program to open them                                                                                                                                              |
| Firewalls            | List of all Firewall rules that are allowed on the host                                                                                                                                                                         |
| IE History           | User Web Browsing History                                                                                                                                                                                                       |
| IE Settings          | User Internet Explorer settings                                                                                                                                                                                                 |
| Installed Programs   | List of all programs that are installed (listed in the uninstall list)                                                                                                                                                          |
| Known DLLs           | List of trusted DLLs that have special treatment by the OS                                                                                                                                                                      |
| LanMan               | List of share permissions and names of folders shared out to the network                                                                                                                                                        |
| Last Visited         | List of executables used by an application to open the files. Also lists the directory location for the last file that was accessed by that application                                                                         |
| Mounted Devices      | This will gather all device that have been mounted on the system (HDD mainly), but will display the connection type, vendor and name                                                                                            |
| Networks             | This is the collection of network information that is gathered from the host. Includes ip4, ip6, any wireless, and list of networks that have been connected to in the past, also NICs connected. (Anything networking related) |
| Office Server Cache  | This is a list of office online locations that a user has accessed (most will be SharePoint)                                                                                                                                    |
| Printers             | This is a list of all printers installed on a computer, not all printers will be added to a user                                                                                                                                |
| Recent Applications  | List of applications that were recently ran per user                                                                                                                                                                            |
| Recent Documents     | This grabs all documents/types that have been opened (up to last 20 in an extension)                                                                                                                                            |
| Recycle Bin Settings | Main item here is the NukeonDelete (if this is set to 1 then recycle bin is not used)                                                                                                                                           |
| Run Programs         | This is list of everything typed into the run command                                                                                                                                                                           |
| Services             | Services that are registered with the OS                                                                                                                                                                                        |
| Shell Folders        | Locations of user’s profile folders (Desktop, Documents, Start Menu, etc.)                                                                                                                                                      |
| User Assist          | List of recent links and executable files that were recently opened per user                                                                                                                                                    |
| User Printers        | This will display the list of any printer that users have added under their profile                                                                                                                                             |
| User List            | List of users, sids, and locations                                                                                                                                                                                              |
| User Shares          | This will list out any and all shares that are setup by the user/system that the user will see.                                                                                                                                 |
| Word Wheel           | Any searches that are done in explorer/start search                                                                                                                                                                             |

I will cover each module more in-depth in the future to do a deep dive into the why I wrote a script to collect the data points in the module.

Next, I wanted to briefly talk about how these scripts get ran. One method will be running them with [Kansa](https://github.com/davehull/Kansa); the output will currently need to be modified to work with this in the current instance of Regscout (A Project for another day). Another method would be to run the scripts directly on a system through PowerShell. As the scripts stand in v. 2.0.1 the scripts run on the windows computer and save the results to data directory in the current location that they have been run from. They all output the data to a JSON file in that data directory and have been aligned to the [Elastic Common Scheme](https://www.elastic.co/guide/en/ecs/current/index.html) to make it easier to work with other tools in Elasticsearch.

The biggest help to running these scripts can be for forensic evidence collection, but I have found that it is a great tool to build an understanding of what is in a network and what could be normal looking usage by the users. Every network is unique from one another due to the habits, job, and abilities of the end users. If you are looking at a bunch of BYOD devices you may see that each device has a list of 20+ wireless SSIDs that it has connected to. Whereas a locked down domain joined workstation it would be very odd to see unusual wireless SSIDs or various other networks in the history of a workstation.

This tool is a great way to build out an understanding of what looks normal in an environment thus allowing analysts to better understand normal; which in turn allows us to find evil.

I hope that this is a great into that will spark an interest in the ability to use the Windows Registry as a great tool for not just a SOC analysist but also for Cyber Security Threat Hunters as well.
