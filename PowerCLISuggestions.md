## Suggestions to make things even Betterer

### Overview:
We compiled some feedback on potential improvements for things ranging from specific cmdlet flexibility and speed to the overall PowerCLI experience (not here in particular order, yet).  If there are other means by which to discuss such feedback in the future, let us do that (not sure this fit into forums / Slack, due to the amount of potential discussion that might ensue).

### For PowerCLI in general:

- support natural PowerShell interactive behavior (tab-completion of possible values based on inventory items in the environment, contributing to discoverability):  add tab-completion support to parameter _values_
	- by registering, for cmdlets where it makes sense, argument completers that provide completion results
	- base/core examples (that might get most use)
		- `-Name` param on things like `Get-Datastore`, `Get-Datacenter`, `Get-Cluster`, `Get-VM`, `Get-VMHost`, `Get-VirtualSwitch`, `Get-VDSwitch`, `Get-Template`
	- go through basic examples from `RegisterArgumentCompleter_samples.ps1`
	- and, like the following, to-be-tab-completed example: `New-VM -Name mattTest_toDelete -Template (Get-Template *2k16<tab>) -ResourcePool (Get-Cluster o<tab>) -Location (Get-Folder v<tab>) -Datastore (Get-DatastoreCluster *gold<tab>) -WhatIf`
	- can be more advanced/contextual, so can take additional pipeline context into consideration.  Like, tab-completing only the VDPG names on given VDSwitch: `Get-VDSwitch myVDSwitch0 | Get-VDPortGroup db<tab>`
	- other examples of where such things are useful:
		- `-NetworkName` on `Set-NetworkAdapter`
		- `-Name` on `Get-VIRole`, `Get-VIPermission`
		- others:  example parameters for which argument completion is useful and that appear in several cmdlets:  `-Location`, `-Datastore`, `-VM`, `-VMHost`, `-Role`, `-Privilege`, `-Tag`, `-Template`

### Thoughts on speed (or slowness) of (what might cause, ways to improve, if any, by environment adjustments):

- `Get-Stat` can be pretty sluggish -- any work being done on speed improvements?
- `Get-ScsiLun` -- ~25s for something like:  `Get-Datastore mydstore0 | Get-ScsiLun`

### Other cmdlet improvement thoughts:

#### Some useful new parameters to add:

- add `-Name` param to `Get-Task`, for more natural getting of task objects (without additional `Where-Object` statement)
- add one or more of `-Name`, `-Key`, `-Label` parameters to `Get-VMHostService`, for getting those services by name/key, instead of a subsequent `Where-Object` to get just the target service(s)
	- like:

	```PowerShell
	## current state
	Get-VMHost myhost0.dom.com | Get-VMHostService | Where-Object {$_.Label -eq "SSH"}

	## Proposed behavior -- more consumer friendly
	Get-VMHost myhost0.dom.com | Get-VMHostService -Name SSH
	Get-VMHost myhost0.dom.com | Get-VMHostService SSH
	```
- to help promote good security practices/behavior, add `-Credential` parameter to `New-VICredentialStoreItem` (and any cmdlets that do not already support it in place of username/password on command line, if there are others)
	- to give option to not enter password in clear on command line (and, to possibly supplant the not-so-great use of `-User` and `-Password`)
- add `-WaitForshutdown` type of parameter to `Stop-VMGuest` so that we easily have code wait for VM to reach powered-off state without write custom "wait" code (or, maybe a `Wait-VMState` kind of cmdlet to allow for waiting for <any> VM powerstate?)
	- for when waiting for updated PowerState property on a VM, for subsequent VM config operations

#### Other thoughts:
- investigate benifits of cmdlet for applying SDRS recommendations -- similar to `Apply-DrsRecommendation`
- add ability to get _effective_ VIPermissions for a principal via `Get-VIPermission`
	- so as to be able to find where userX has access at all, not just explicit VIPermissions granted (like, for a member of an AD group)
	- say, via a switch param like `-Effective`, so that this useful permissions retrieval, which would likely be more resource intensive, is not the default behavior
- add consideration for `DefaultIntraVmAffinity` property on datastoreclusters
	- this is the property of a datastorecluster that controls behavior that, if enabled, tries to keep all of a given VM's disks on the same datastore in the given datastorecluster
	- add ability to set it via `Set-DatastoreCluster`
	- add some top-level property to `VMware.VimAutomation.ViCore.Impl.V1.DatastoreManagement.DatastoreClusterImpl` objects, so one need not dig down to `ExtensionData.PodStorageDrsEntry.StorageDrsConfig.PodConfig.DefaultIntraVmAffinity` property to get current value
- add ability to retrieve the types of events like the way `Get-StatType` gets types of stats available for entities
	- maybe via addition of `Get-VIEventType` kind of cmdlet
	- and then the ability to provide such types as a parameter to `Get-VIEvent` to retrieve only desired types of events, like:
	```PowerShell
	## Proposed capability to get only ssh disable events
	Get-VIEvent -EventType esx.audit.ssh.disabled
	```
	- accepting event type parameter is currently only achieved by advanced functions that may support it, like `Get-VIEventAdvanced.ps1`

- add `FullPath` type of property to returned `VMware.VimAutomation.ViCore.Impl.V1.Inventory.FolderImpl` objects
- add ability to have a Cluster object be a search root (via `-Location` param) for getting templates via `Get-Template`, in the same way that `Get-VM` accepts a cluster object for `-Location`; removes need to have `Get-VMHost` in the mix when just wanting to check a cluster for the presence of a template
	```PowerShell
	## current state
	Get-Cluster myCluster0 | Get-VM myVM0
	Get-Cluster myCluster0 | Get-VMHost | Get-Template myTemplate0 ## <-- behavior to improve

	## Proposed behavior -- more consumer friendly -- can use Get-Template in same way as Get-VM
	Get-Cluster myCluster0 | Get-Template myTemplate0
	```
- easy win:  update default table view (say, in the corresponding *.format.ps1xml) to be more informative for object `VMware.VimAutomation.ViCore.Impl.V1.Host.Storage.VMHostStorageInfoImpl` (display more than a single property of the object in default table view)
	- this is the returned object from things like `Get-VMHostStorage`, often used for rescanning HBAs
	- would be useful for consumer to have more information by default on what is the VMHost to which the given VMHostStorageInfo object pertains (currently, only the `SoftwareIScsiEnabled` property is displayed in the default table view)
	```PowerShell
	## current state
	Get-Cluster myCluster0 | Get-VMHost | Get-VMHostStorage -RescanAllHba
	
	SoftwareIScsiEnabled
	--------------------
	False

	## Proposed behavior -- more consumer friendly:  more informative default table view, displaying a couple of other existing properties of the returned object
	Get-Cluster myCluster0 | Get-VMHost | Get-VMHostStorage -RescanAllHba
	
	VMHost                SoftwareIScsiEnabled ScsiLun
	------                -------------------- -------
	myhost0.dom.com                      False {naa.60060160...}
	```
- VCSA VAMI API:  if not already available, add ability to check for / apply / install appliance updates and patches; already available?
	- currently have seen Set and Get for the "Update repository configuration", but have found no other items in APIExplorer
