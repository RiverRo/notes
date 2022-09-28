# WSL2

Ubuntu WSL2 implementation via Visual Studio Code in Windows 10/11

> [Main Table of Contents](../README.md)

## In This File

- cron, crontab
  - nano
  - cron
  - crontab
  - configure pc
  - configure python program into executable script
  - generic pc sleep batch (.bat) file for windows
  - windows task scheduler

## Cron, Crontab

- The Ubuntu WSL2 via visual studio code in Windows does NOT automatically start cron
- Use Windows Task Scheduler to run WSL and cron on a schedule
- As of 10/17/2022 only use sudo version of cron and crontab because the non-sudo version doesn't work for me
- Cron is a daemon service that works off a schedule
- Crontab is the schedule
- Choose nano text editor (easiest) to edit files

  ### nano

  ```python
  Save File: Ctrl+o then Enter
  Exit nano: Ctrl+x
  ```

  ### cron

  ```python
  # Check if cron is installed
  which cron


  # Check if cron daemon is running
  sudo service cron status

  # Start cron daemon
  sudo service cron start

  # Stop cron daemon
  sudo service cron stop


  # The above commands require passwords
  # In order to use cron with Windows Task Scheduler must bypass this password by going into:
  sudo visudo
  # Add these to the last lines of the file
  %sudo ALL=NOPASSWD: /usr/sbin/service cron start
  # Add empty line here
  %sudo ALL=NOPASSWD: /usr/sbin/service cron stop
  # Add empty line here as well
  ```

  ### crontab

  - [crontab guru](https://crontab.guru/)
  - [crontab gotchas](https://serverfault.com/questions/449651/why-is-my-crontab-not-working-and-how-can-i-troubleshoot-it)
  - [crontab gotchas more](https://askubuntu.com/questions/23009/why-crontab-scripts-are-not-working)
  - [crontab deprecated in macos](https://alvinalexander.com/mac-os-x/mac-osx-startup-crontab-launchd-jobs/)

  ```python
  # Edit crontab schedule
  sudo crontab -e

  # Example schedule used for a scraper to run every day at 2am local time
  # This file must be set to an executable with: chmod +x filename
  # Must add a shebang to the file pointing to a python program
  # The absolute path name must be used here in crontab and within the program, no relative file paths.
  0 2 * * * /home/sportybutton/projects/scraper_btmbrk/app.py  # Then press enter for empty line
  # Gotcha: This comment line must be an empty line in the real file for cron to properly run
  ```

  ### configure python program into executable script

  - The file that is being scheduled to run must be configured
  - File must be set to an executable with: chmod +x filename
  - Must add a shebang to the file pointing to a python program
  - Use absolute path names, no relative path names
  - Must add this line after the imports section
    ```python
    BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
    sys.path.append(BASE_DIR)
    ```
  - If the above don't work, [must further reconfigure file as a script](https://cronitor.io/cron-reference/python-cron-jobs)

  ### configure pc

  - Allow waker timers

        	In PC
        	1) control panel -> Hardware and Sounds -> Power Options -> change plan settings
        	            -> change advanced power   settings -> Sleep -> Allow wake timers
        	2) Battery: Enable and Plugged in: Enable

  - Windows has hibernate/sleep. Disable hibernate to automatically sleep when executing a sleep batch (.bat) file

    - Configure windows sleep/hibernate states [disable hibernate reference](https://www.groovypost.com/howto/schedule-wake-sleep-windows-automatically/)

          In PC
          1) Start button
          2) Find cmd.exe and right-click to run as administraer
          3) Type: powercfg -h off

  ### generic pc sleep batch (.bat) file for windows

  1.  Open notepad
      ```python
      # e.g. pc_sleep.bat
      Rundll32.exe Powrprof.dll,SetSuspendState Sleep
      ```
  2.  Save this file with .bat extension e.g. pc_sleep.bat anywhere in Windows file system (not WSL file system)
  3.  Save as type: `all files`
  4.  NOTE: I've read rundll32 has been deprecated, if above stops working try this more [robust sleep script](https://www.addictivetips.com/windows-tips/schedule-sleep-on-windows-10/)

      ```
      @echo off &mode 32,2 &color cf &title Power Sleep
      set "s1=$m='[DllImport ("Powrprof.dll", SetLastError = true)]"
      set "s2=static extern bool SetSuspendState(bool hibernate, bool forceCritical, bool disableWakeEvent);"
      set "s3=public static void PowerSleep(){ SetSuspendState(false, false, false); }';"
      set "s4=add-type -name Import -member $m -namespace Dll; [Dll.Import]::PowerSleep();"
      set "ps_powersleep=%s1%%s2%%s3%%s4%"
      call powershell.exe -NoProfile -NonInteractive -NoLogo -ExecutionPolicy Bypass -Command "%ps_powersleep:"=\"%"
      exit
      ```

  ### windows task scheduler

  - [How to launch cron automatically in WSL on Windows reference](https://www.howtogeek.com/746532/how-to-launch-cron-automatically-in-wsl-on-windows-10-and-11/)

  - All Tasks

    - Top ribbon -> Action -> Create Task
    - General Tab
      - Name: Add any name
      - Check 'Run whether user is logged on or not'
      - Check 'Run with highest privileges'
      - Configure for: Windows 10
    - Triggers Tab
      - New, set start date, time and schedule
    - Actions Tab
      - New
      - Action: Start a program
      - Settings section: See below for different programs
    - Conditions Tab
      - Check 'Wake the computer to run this task' (if it's not a sleep program)
      - Uncheck everything if sleep program
    - Settings Tab
      - Check 'Allow task to be run on demand'
      - Check 'Stop the task if it runs longer than 1hr'
      - check 'If the running task does not end when requesed, force it to stop'

  - Examples Specific Program Settings

    - Wake PC up from sleep, Start WSL2 and cron - Conditions Tab

      1.  Check 'Wake the computer to run this task'

    - Wake PC up from sleep, Start WSL2 and cron - Actions Tab

      1.  Action: Start a program
      2.  Program/Script: `C:\Windows\System32\wsl.exe`
      3.  Add arguements: `sudo /usr/sbin/service cron start`

    - Stop cron and put PC to sleep - Conditions Tab

      1.  Uncheck everything

    - Stop cron and put PC to sleep - TWO Actions Tab

      - First Action - stop cron

        1.  Action: Start a program
        2.  Program/Script: `C:\Windows\System32\wsl.exe`
        3.  Add arguements: `sudo /usr/sbin/service cron stop`

      - Second Action - put PC to sleep

        1.  Action: Start a program
        2.  Program/Script: `locate path to sleep batch (.bat) file` in windows file system (not WSL file system)

## DOCUMENT STYLES

<style>
h1 {
  color: DeepSkyBlue;
}
h2 {
color: yellow;
}
h3 {
  color: LightCoral;
}
</style>
