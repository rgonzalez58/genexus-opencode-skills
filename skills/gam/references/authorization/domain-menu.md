---
name: domain-menu
description: GAMApplication menu system — permission-filtered navigation, menu and option CRUD, GoHome
---

# GAM Application Menu System
GAM includes a built-in menu system that resolves navigation options based on the authenticated user's permissions. Menus are managed through the `GAMApplication` External Object

Related files:
- [GAM Authorization — Permission Evaluation](domain-authz.md) — permission evaluation (menu resolution depends on it)
- [GAMRole Code Provider](../kb-setup/init/code-provider/gamrole-code-provider.md) — permission and role CRUD
- [GAMApplication Code Provider](../kb-setup/init/code-provider/gamapplication-code-provider.md) — Application entity management
- [GAM Backoffice — Applications (GAM_Applications)](../backoffice/applications.md) — Backoffice UI for menus and options

For the complete interface, inspect the `GAMApplication` External Object directly in the KB

---

## Behavior
- An Application has one or more Menus, each containing Menu Options
- Each Menu Option can be linked to a Permission — only users with that permission see the option
- `GetUserMenu` / `GetUserMainMenu` resolve the menu for the current user, filtering out options the user lacks permission to see
- `GoHome()` redirects to the Application's configured home object
- Menu resolution is repository-scoped — menus from Repository A are not visible in Repository B

## Menu CRUD (critical operations)
Add and read a menu:

```genexus
&Ok = GAMApplication.AddMenu(&Menu, &Errors)
&Menu = GAMApplication.GetMenu(&MenuId, &Errors)
```

List and navigate menus:

```genexus
&Menus = GAMApplication.GetMenus(&Filter, &Errors)
&SubMenus = GAMApplication.GetSubMenus(&ParentMenuId, &Errors)
```

Full CRUD surface (update, delete) is available on the `GAMApplication` External Object. Pattern follows the same `(&entity, &Errors)` convention

## Menu Options CRUD
Add and read an option:

```genexus
&Ok = GAMApplication.AddMenuOption(&MenuId, &Option, &Errors)
&Option = GAMApplication.GetMenuOption(&MenuId, &OptionId, &Errors)
```

List options under a menu:

```genexus
&Options = GAMApplication.GetMenuOptions(&MenuId, &Filter, &Errors)
```

Update and delete methods follow the same pattern on `GAMApplication`

## User Menu Resolution (Permission-Filtered)
Get the authenticated user's main menu:

```genexus
&AdditionalParameters = new()
&UserMainMenu = GAMApplication.GetUserMainMenu(&AdditionalParameters, &Errors)
If &Errors.Count > 0
	Return
EndIf
```

Resolve by Id or GUID:

```genexus
&UserMenu = GAMApplication.GetUserMenu(&MenuId, &AdditionalParameters, &Errors)
&UserMenu = GAMApplication.GetUserMenuByGUID(&MenuGUID, &AdditionalParameters, &Errors)
```

## Navigation
```genexus
&HomeObject = GAMApplication.HomeObject
GAMApplication.GoHome()
```

## Permission Resources
List all available resources for permission assignment:

```genexus
&Resources = GAMApplication.GetPermissionResources(&Filter, &Errors)
```

## Pattern: Authorize a Menu Option
```genexus
&AdditionalParameters = new()
&UserMainMenu = GAMApplication.GetUserMainMenu(&AdditionalParameters, &Errors)
If &Errors.Count > 0
	msg(!"User menu resolution failed")
	Return
EndIf

&MainMenuId = &UserMainMenu.Id
&TargetOptionId = !"CreateOrder"
&TargetOption = GAMApplication.GetMenuOption(&MainMenuId, &TargetOptionId, &Errors)
If &Errors.Count > 0 or &TargetOption.IsEmpty()
	msg(format(!"Access denied: %1 option unavailable", &TargetOptionId))
	GAMApplication.GoHome()
	Return
EndIf
```

## Constraints
- Menu options only appear for users with the corresponding permission granted
- `GetUserMainMenu` requires an active authenticated session
- `GoHome()` redirects to the home object configured for the Application context
- Menu resolution is repository-scoped
