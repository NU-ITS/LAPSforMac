# LAPSforMac
Local Administrator Password Solution for Mac

## Purpose  

We needed a way to securely manage local admin accounts on our Macs so we developed this system to complement Microsft LAPS, used by our Windows colleagues. As currently designed, this solution creates a local Admin account on every Mac enrolled into Casper and stores the account password in the Mac's inventory record as an Extension Attribute. On a specified interval Casper will then randomize the local Admin account password going forward.  

As written currently, LAPS has several components that are integrated with the JAMF Casper Suite:

1.  A Casper Computer Extension Attribute to hold the current LAPS password.
2.  A Casper local user account for API calls.
3.  Two Smart Groups used to identify Computers with/without the local admin account.
4.  The LAPS Account Creation script.
5.  The LAPS script.
6.  A Casper policy that creates the local Account, via a manual trigger.
7.  A Casper policy that creates the local Account, for FileVault enabled Macs, via a manual trigger.
8.  A Casper policy that randomizes the Local Admin account password on a specified interval, by running a script.
9.  A Casper policy that calls LAPS Account Creation script.
10. A local log for LAPS on each Mac.

## Admin Defined Variables
```{APIusername}```  
This is the name of the local user within Casper that will be leveraged by the API.  
   
```{APIpassword}```  
This is the password used by the Local User within Casper, it should be long and robust.  
   
```{AccountShortName}```  
This is the shortname of your Local Admin account that will be created on your client Macs enrolled in Casper.  
   
```{AccountDisplayName}```  
This is the display name of your Local Admin account that will be created on your client Macs enrolled in Casper.  
   
```{AccountInitialPassword}```  
This will be the seed password for creating your Local Admin account on your Macs. This is required to maintain a functional FileVault and keychain for the account. This password is immediately randomized after the account is created.  

# Component Setup

## 1. Casper Computer Extension Attribute

    Display Name: LAPS (This name is hardcoded into the scripts, if you change this name update the scripts accordingly)  
    Description: This attribute will display the current Local Admin Password of the device.  
    Data Type: String  
    Inventory Display: General  
    Input Type: Text Field  
    Recon Display: User and Location (Not Used)  

*Notes: The field is editable to allow for troubleshooting or manually overriding the password.*

## 2. Casper API User

    Username: {APIusername}
    Access Level: Full Access
    Privilege Set: Custom
    Access Status: Enabled
    Full Name: {APIusername}
    Email Address: (Not Used)
    Password: {APIpassword}
    Privileges:
		JSS Objects:
			Computer Extension Attributes: RU
			Computers: RU
			Users: U

*Notes: For Casper permissions C=Create, R=Read, U=Update, D=Delete (Not sure why the "Users" permission is needed. After much trial and error, and a call to JAMF, I discovered this permission set was required to properly read and update the Computer tables)*

## 3. Casper Smart Groups
Replace ```{AccountShortName}``` with the name of the local admin account you will use for LAPS.

	1. Display Name: {AccountShortName} LAPS User Missing
		Criteria: Local User Accounts, does not have, {AccountShortName}

	2. Display Name: {AccountShortName} LAPS User Present
		Criteria: Local User Accounts, has, {AccountShortName}



## 4. LAPS Account Creation script
    Display Name: LAPS Account Creation
    Options:
    	Priority: Before
    	Parameter Labels:
    		Parameter 4: API Username
			Parameter 5: API Password
			Parameter 6: LAPS Account Shortname
			Parameter 7: LAPS Account Display Name
			Parameter 8: LAPS Password Seed
			Parameter 9: LAPS Account Event
			Parameter 10: LAPS Account Event FVE
			Parameter 11: LAPS Run Event
### Script
The current version of the LAPS Account Creation script is available [here](https://github.com/unl/LAPSforMac/blob/master/LAPS%20Account%20Creation.sh).

*Notes: The LAPS Account Creation script performs the following actions:*  

```
1. Verifies that all variable parameters have been populated within Casper.  
2. Verifies the location of the JAMF binary.  
3. Populates the Local Admin account password seed into the LAPS extension attribute within Casper.  
4. Checks if FileVault 2 in enabled on the Mac then calls Casper to create the local admin account accordingly. 
	• If FileVault 2 is not enabled, a regular admin account will be created on the Mac.
	• If FileVault 2 is enabled, a FileVault 2 enabled admin account will be created on the Mac, the script will then verify that the new admin account is listed as FileVault enabled.  
5. After the account has been created the LAPS script is called to randomize the initial password seed.
```
### Variables
```apiURL``` Put the fully qualified domain name address of your Casper server, including port number  
*(Your port is usually 8443 or 443; change as appropriate for your installation)*

```LogLocation``` Put the preferred location of the log file for this script. If you don't have a preference, using the default setting of ```/Library/Logs/Casper_Laps.log``` should be fine.



## 5. LAPS script
	Display Name: LAPS
	Options:
	Priority: After
	Parameter Labels:
		Parameter 4: API Username
		Parameter 5: API Password
		Parameter 6: LAPS Account Shortname
		
### Script
The current version of the LAPS script is available [here](https://github.com/unl/LAPSforMac/blob/master/LAPS.sh).

*Notes: The LAPS script performs the following actions:*  

```
1. Verifies that all variable parameters have been populated within Casper.  
2. Verifies the location of the JAMF binary.  
3. Verifies that a password is stored in the LAPS extension attribuite within Casper for this Mac.
	• If no password is found or it is invalid, the script will proceed with a brute force reset of the password.
	• If a password is valid, the script will reset the password and update the local Keychain and FileVault 2.
4. After reseting the password the script will then update the LAPS extension attribute for the Mac in Casper and verify that the new entry in Casper is valid on the local Mac.

```


### Variables
```apiURL``` Put the fully qualified domain name address of your Casper server, including port number  
*(Your port is usually 8443 or 443; change as appropriate for your installation)*

```LogLocation``` Put the preferred location of the log file for this script. If you don't have a preference, using the default setting of ```/Library/Logs/Casper_Laps.log``` should be fine.  

```newPass``` This function controls the randomized password string. If you don't have a preference, the default should be fine for your environment.

*The diagram below details how the newPass function works, if you wish to modify the password string.*

			   ┌─── openssl is used to create 
			   │	  a random Base64 string
			   │				      ┌── remove ambiguous characters
			   │			          │
	┌──────────┴──────────┐	  ┌───┴────────┐
	openssl rand -base64 10 | tr -d OoIi1lLS | head -c12;echo
											   └──────┬─────┘
											   		  │
	        	prints the first 12 characters	──────┘
	          	of the randomly generated string

		
				
## 6. Casper LAPS Account Creation Policy
	Display Name: LAPS for {AccountShortName} – Create Local Account – Manual Trigger
	Scope: All Computers
	Trigger:
		Custom: createLAPSaccount-{AccountShortName}
	Frequency: Ongoing
	Local Accounts:
		Action: Create Account
		Username: {AccountShortName}
		Full Name: {AccountDisplayName}
		Password: {AccountInitialPassword}
		Verify Password: {AccountInitialPassword}
		Home Directory Location: /Users/{AccountShortName}/
		Password Hint: (Not Used)
		Allow user to administer computer: Yes
		Enable user for FileVault 2: No	
## 7. Casper LAPS Account Creation Policy for FileVault 2 Enabled Macs
This is a separate policy to eliminate false positve errors that accumulate in the logs if the Mac is using FileVault 2.

	Display Name: LAPS for {AccountShortName} – Create Local Account FVE – Manual Trigger
	Scope: All Computers
	Trigger:
		Custom: createLAPSaccountFVE-{AccountShortName}
	Frequency: Ongoing
	Local Accounts:
		Action: Create Account
		Username: {AccountShortName}
		Full Name: {AccountDisplayName}
		Password: {AccountInitialPassword}
		Verify Password: {AccountInitialPassword}
		Home Directory Location: /Users/{AccountShortName}/
		Password Hint: (Not Used)
		Allow user to administer computer: Yes
		Enable user for FileVault 2: Yes
## 8. Casper LAPS Policy
This policy randomizes the local admin accounts password on a specified interval.

	Display Name: LAPS for {AccountShortName}
	Scope: LAPS {AccountShortName} Account Present
	Trigger: Recurring Check-in
	Frequency: Once every day (Change this value to meet your institution's needs)
	Scripts: LAPS
		Priority: After
		Parameter Values
			API Username: {APIusername}
			API Password: {APIpassword}
			LAPS Account Shortname: {AccountShortName}
##9. Casper policy to call the LAPS Account Creation script.
	Name: LAPS – Create Account
	Scope: {AccountShortName} LAPS Account Missing
	Trigger: Startup, Check-in, Enrollment (You may also decide to add a manual trigger for advanced workflows)
	Frequency: Ongoing
	Scripts: LAPS Account Creation
		Priority: Before
		Parameter Values
			API Username: {APIusername}
			API Password: {APIpassword}
			LAPS Account Shortname: {AccountShortName}
			LAPS Account Display Name: {AccountDisplayName}
			LAPS Password Seed: {AccountInitialPassword}
			LAPS Account Event: createLAPSaccount-{AccountShortName}
			LAPS Account Event FVE: createLAPSaccountFVE-{AccountShortName}
			LAPS Run Event: runLAPS
## 10. LAPS Log
A log is written to each Mac run LAPS for troubleshooting. The default location for this log is ```/Library/Logs/Casper_LAPS.log``` which can be modified if desired.
