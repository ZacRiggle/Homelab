# Day 2 — OUs, Users, Security Groups & Wallpaper GPO

**Date:** 2026-03-27

## What I Did

### Created Organizational Unit Structure

Opened Active Directory Users and Computers (ADUC) on DC01 via Server Manager → Tools. The domain already had default containers (Builtin, Computers, Users, etc.) that are created automatically when you promote a DC. These are containers, not OUs, so you can't link Group Policies to them — which is why I built my own OU structure.

Right-clicked `ADLab.local` → New → Organizational Unit:

```
ADLab.local
├── HR
├── IT Department
├── Sales
├── Servers
├── Workstations
└── ALL-USERS (Security Group)
```

All OUs sit directly under ADLab.local. Moved CLIENT01 from the default Computers container into the **Workstations** OU by right-clicking → Move. Left DC01 in the default Domain Controllers OU — domain controllers should never be moved from there because critical default GPOs are linked to it.

![ADUC showing the full OU structure under ADLab.local](./screenshots/aduc-ou-structure.png)

### Created User Accounts

Created users inside their respective department OUs:

| Username | Full Name | OU |
|----------|-----------|-----|
| zac r | Zac R | IT Department |
| james jr. r | James Jr. R | IT Department |
| dannelle r | Dannelle R | IT Department |
| kam r | Kam R | HR |
| pib a | Pib A | HR |
| sammy d | Sammy D | HR |
| madi pl | Madi Pl | Sales |
| mandy a | Mandy A | Servers |
| andy r | Andy R | Workstations |

Created via: Right-click on the OU → New → User. Set passwords, unchecked "User must change password at next logon" for lab convenience.

### Created Security Groups

Created security groups to manage permissions by role rather than assigning access to individual users.

| Group Name | Location | Scope/Type | Members |
|------------|----------|------------|---------|
| IT-ADMIN | IT Department | Global / Security | james jr. r, zac r |
| HR-ADMIN | HR | Global / Security | kam r, pib a |
| ALL-USERS | ADLab.local (root) | Global / Security | andy r, dannelle r, james jr. r, kam r, madi pl, mandy a, pib a, sammy d, zac r |

ALL-USERS sits at the domain root level since it spans all departments. IT-ADMIN and HR-ADMIN sit inside their respective OUs.

![IT Department OU showing users and IT-ADMIN group with members](./screenshots/it-department-users-and-group.png)

![ALL-USERS security group showing all domain members](./screenshots/all-users-group-members.png)

### Created Desktop Wallpaper GPO

Set up a network share for the wallpaper image:
1. Created `C:\Wallpaper` on DC01
2. Placed `DSWP.jpg` in the folder
3. Shared the folder as `Wallpaper` with read access for domain users

Created the GPO:
1. Opened Group Policy Management on DC01
2. Right-clicked the **Company** OU → "Create a GPO in this domain, and Link it here"
3. Named it `Desktop Wallpaper`
4. Edited → User Configuration → Policies → Administrative Templates → Desktop → Desktop
5. Enabled **Desktop Wallpaper**
6. Wallpaper Name: `\\DC01\Wallpaper\DSWP.jpg`
7. Wallpaper Style: Fill

![GPO editor showing Desktop Wallpaper policy enabled with UNC path](./screenshots/gpo-desktop-wallpaper-settings.png)

### Tested GPO on CLIENT01

Logged into CLIENT01 as a domain user, ran `gpupdate /force`, then logged out and back in. The Death Star wallpaper appeared on the desktop — GPO applied successfully. Users can't change the wallpaper since the policy overrides the setting.

![CLIENT01 desktop showing the wallpaper applied via Group Policy](./screenshots/client01-wallpaper-applied.png)

## What I Learned

- The default containers (Builtin, Computers, Users) are not OUs — you can't link GPOs to them, which is why creating your own OU structure is important
- Never move domain controllers out of the Domain Controllers OU — it has critical default policies
- Security groups are how permissions scale — in a real environment you assign access to groups, never individual users
- Group scope "Global" is the standard for department and role-based groups
- Wallpaper GPO uses a UNC path (`\\DC01\Wallpaper\...`) so every domain machine pulls the image from the server share — this way you update the wallpaper in one place and it changes everywhere
- User Configuration policies like wallpaper only apply at logon, not during a background `gpupdate` — you have to log out and back in to see the change
