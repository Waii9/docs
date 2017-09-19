---
title: Using Rules with the Authorization Extension
description: How to use information from the extension in rules
---

# Authorization Extension: Rules

With [rules](/rules) that execute during the login process, you can grant them permissions to reach out to the Authorization Extension to do things like:

1. Determining the user's group membership, roles and permissions
2. Storing the user's groups, roles and permissions info as [part of the `app_metadata`](/extensions/authorization-extension/v2/configuration#persistence)
3. Adding the user's groups, roles and permissions to the [outgoing token]((/extensions/authorization-extension/v2/configuration#token-contents)) (which can be requested via the `openid groups permissions roles` scope)

Because the above logic is part of a rule, it will only be executed in the context of a login. If users are added to or removed from a group, this change will only be reflected in Auth0 after the user's next login.

## Control App Access

You can also write rules that are executed after the Authorization Extension rule to do things like control access to your application. One method of doing this is to specify the roles that are required for each client using the [client metadata](/rules/metadata-in-rules#reading-metadata).

### Step 1: Set the Client Metadata's `required_roles`

⁠⁠⁠⁠1. To set the `context.clientMetadata` field with `required_roles`, begin by selecting the client you want to work with [in the dashboard](${manage_url}/#/clients). 

This brings you to the client's **Settings**. Scroll down and click **Show Advanced Settings** at the bottom of the page.

![Click Advanced Settings Link](/media/articles/extensions/authorization/adv-settings-link.png)

2. Under **Client Metadata** add an item setting the **Key** to `required_roles` and in **Value** field list your roles in comma separated style. Click the **CREATE** button to add the field.

![Example of required roles](/media/articles/extensions/authorization/required-roles.png)

3. When finished click **Save Changes**. Now when you login from this client, in `context.clientMetadata` you will have the `required_roles` with the roles value string you entered.

### Step 2: Create the Rule Enforcing Client Roles

::: warning
Before creating this rule, enable **Roles** under the [Token Contents](/extensions/authorization-extension/v2/configuration#token-contents) and [publish the Authorization Extension rule](/extensions/authorization-extension/v2/configuration#publish-the-authorization-extension-rule. Then, add this rule and make sure it is listed *after* the generated "auth0-authorization-extension" rule.
:::

After setting `required_roles`, create a new [rule](${manage_url}/#/rules) with the following body:

```js
function (user, context, callback) {
  context.clientMetadata = context.clientMetadata || {};
  if (context.clientMetadata.required_roles && context.clientMetadata.required_roles.length){
    if (user.roles) {
      var _ = require('lodash');
      var roles = context.clientMetadata.required_roles.split(',');
      var matchingRoles =_.filter(user.roles, function(roleName) {
        return _.includes(roles, roleName);
      });

      if (matchingRoles && matchingRoles.length) {
        return callback(null, user, context);
      }
    }

    return callback(new UnauthorizedError('You do not have the required role to access ' + context.clientName));
  }

 callback(null, user, context);
}
```