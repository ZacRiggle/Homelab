Day 2 — OUs, Users, Security Groups & Wallpaper GPO
Date: 2026-03-27
What I Did
Created Organizational Unit Structure
Opened Active Directory Users and Computers (ADUC) on DC01 via Server Manager → Tools. The domain already had default containers (Builtin, Computers, Users, etc.) that are created automatically when you promote a DC. These are containers, not OUs, so you can't link Group Policies to them — which is why I built my own OU structure.
Right-clicked ADLab.local → New → Organizational Unit:
ADLab.local
├── HR
├── IT Department
├── Sales
├── Servers
├── Workstations
└── ALL-USERS (Security Group)
All OUs sit directly under ADLab.local. Moved CLIENT01 from the default Computers container into the Workstations OU by right-clicking → Move. Left DC01 in the default Domain Controllers OU — domain controllers should never be moved from there because critical default GPOs are linked to it.
Created User Accounts
Created users inside their respective department OUs:
UsernameFull NameOUzac rZac RIT Departmentjames jr. rJames Jr. RIT Departmentdannelle rDannelle RIT Departmentkam rKam RHRpib aPib AHRsammy dSammy DHRmadi plMadi PlSalesmandy aMandy AServersandy rAndy RWorkstations
Created via: Right-click on the OU → New → User. Set passwords, unchecked "User must change password at next logon" for lab convenience.
Created Security Groups
Created security groups to manage permissions by role rather than assigning access to individual users.
Group NameLocationScope/TypeMembersIT-ADMINIT DepartmentGlobal / Securityjames jr. r, zac rHR-ADMINHRGlobal / Securitykam r, pib aALL-USERSADLab.local (root)Global / Securityandy r, dannelle r, james jr. r, kam r, madi pl, mandy a, pib a, sammy d, zac r
ALL-USERS sits at the domain root level since it spans all departments. IT-ADMIN and HR-ADMIN sit inside their respective OUs.
Created Desktop Wallpaper GPO
Set up a network share for the wallpaper image:

Created C:\Wallpaper on DC01
Placed DSWP.jpg in the folder
Shared the folder as Wallpaper with read access for domain users

Created the GPO:

Opened Group Policy Management on DC01
Right-clicked the Company OU → "Create a GPO in this domain, and Link it here"
Named it Desktop Wallpaper
Edited → User Configuration → Policies → Administrative Templates → Desktop → Desktop
Enabled Desktop Wallpaper
Wallpaper Name: \\DC01\Wallpaper\DSWP.jpg
Wallpaper Style: Fill

Tested GPO on CLIENT01
Logged into CLIENT01 as a domain user, ran gpupdate /force, then logged out and back in. The Death Star wallpaper appeared on the desktop — GPO applied successfully. Users can't change the wallpaper since the policy overrides the setting.
What I Learned

The default containers (Builtin, Computers, Users) are not OUs — you can't link GPOs to them, which is why creating your own OU structure is important
Never move domain controllers out of the Domain Controllers OU — it has critical default policies
Security groups are how permissions scale — in a real environment you assign access to groups, never individual users
Group scope "Global" is the standard for department and role-based groups
Wallpaper GPO uses a UNC path (\\DC01\Wallpaper\...) so every domain machine pulls the image from the server share — this way you update the wallpaper in one place and it changes everywhere
User Configuration policies like wallpaper only apply at logon, not during a background gpupdate — you have to log out and back in to see the change