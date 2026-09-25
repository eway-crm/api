# Breaking Changes

## 10.1

Procedure `eWaySP_SendMail` raises error when legacy `@format` parameter is provided, you should remove it from the procedure calls.

## 9.3

### Renamed database objects

Several database objects were renamed.

#### Stored procedures:

* `eWaySP_AssingUserTeamRole` -> `eWaySP_AssignUserTeamRole`
* `eWaySP_CopyAddtionalFieldsData` -> `eWaySP_CopyAdditionalFieldsData`
* `eWaySP_ReasignTasks` -> `eWaySP_ReassignTasks`
* `eWaySP_SetAdminGroupModulePermisison` -> `eWaySP_SetAdminGroupModulePermission`
* `eWaySP_SetColumnPermissionForPeopleExpensesSumarization` -> `eWaySP_SetColumnPermissionForPeopleExpensesSummarization`
* `eWaySP_SetColumnPermissionForPriceSumarization` -> `eWaySP_SetColumnPermissionForPriceSummarization`
* `eWaySP_UpdateWorkReportsByCoutryOfUser` -> `eWaySP_UpdateWorkReportsByCountryOfUser`

#### Functions:

* `ConcateEmailAddresses` -> `ConcatenateEmailAddresses`
* `GetDatesDifferenceInBussinesHours` -> `GetDatesDifferenceInBusinessHours`

## 8.1

### ItemChanges

ItemVersion column in EWD_ItemChanges became required.

When there are records with NULL in EWD_ItemChanges table during upgrade, procedure **eWaySP_TruncateItemChanges** is called
and all records in EWD_ItemChanges table are deleted. ChangeID is reset and starts with 1.

Starting with version 8.1 you may notice the following error in your triggers:

```
Cannot insert the value NULL into column 'ItemVersion', table 'dbo.EWD_ItemChanges'
```

When saving single item make sure to call **eWaySP_InsertIntoItemChanges** procedure. It will automatically fill ItemVersion.

When saving multiple items use our [Update multiple items snippet](https://github.com/eway-crm/Snippets).

### RTFBody / TeamTask

We have removed columns `RTFBody` from `EWD_Tasks` and `EWD_Calendar` tables as they have not been used for long time.
Also `TeamTask` column was removed from `EWD_Tasks` table.
