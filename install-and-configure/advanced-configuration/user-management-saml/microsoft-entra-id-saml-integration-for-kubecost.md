# Microsoft Entra ID SAML Integration for nOps

{% hint style="info" %}
SSO and RBAC are only officially supported on nOps Enterprise plans.
{% endhint %}

This guide will show you how to configure nOps integrations for SAML and RBAC with Microsoft Entra ID.

## Entra ID SAML configuration

### Step 1: Create an enterprise application in Microsoft Entra ID for nOps

1. In the [Azure Portal](https://portal.azure.com/#view/Microsoft\_AAD\_IAM/ActiveDirectoryMenuBlade/\~/Overview), go to the Microsoft Entra ID Overview page and select _Enterprise applications_ in the left navigation underneath Manage.
2. On the Enterprise applications page, select _New application._
3. On the Browse Microsoft Entra ID Gallery page, select _Create your own application_ and select _Create._ The 'Create your own application window' opens.
4. Provide a custom name for your app. Then, select _Integrate any other application you don't find in the gallery._ Select _Create_.

### Step 2: Configuring an Entra ID Enterprise Application

1. Return to the Enterprise applications page from Step 1.2. Find and select your Enterprise application from the table.
2. Select _Properties_ in the left navigation under Manage to begin editing the application. Start by updating the logo, then select _Save_. Feel free to use an [official nOps logo](https://github.com/nOps/poc-common-configurations/blob/main/saml-azuread/images/nOps-logo.png).
3. Select _Users and groups_ in the left navigation. Assign any users or groups you want to have access to nOps, then select _Assign._
4. Select _Single sign-on_ from the left navigation. In the 'Basic SAML Configuration' box, select _Edit_. Populate both the Identifier and Reply URL with the URL of your nOps environment _without a trailing slash_ (ex: http://localhost:9090), then select _Save_. If your application is using OpenId Connect and OAuth, most of the SSO configuration will have already been completed.

<figure><img src="/.gitbook/assets/okta-saml-1.png" alt=""><figcaption></figcaption></figure>

5. (Optional) If you intend to use RBAC, you also need to add a group claim. Without leaving the SAML-based Sign-on page, select _Edit_ next to Attributes & Claims. Select _Add a group claim._ Configure your group association, then select _Save_. The claim name will be used as the `assertionName` value in the [_values-saml.yaml_](https://github.com/nOps/poc-common-configurations/blob/main/saml-azuread/values-saml.yaml) file.
6. On the SAML-based Sign-on page, in the SAML Certificates box, copy the login of 'App Federation Metadata Url' and add it to your _values-saml.yaml_ as the value of `idpMetadataURL`.

<figure><img src="/.gitbook/assets/okta-saml-2.png" alt=""><figcaption></figcaption></figure>

7. In the SAML Certificates box, select the _Download_ link next to Certificate (Base64) to download the X.509 cert. Name the file _myservice.cert_.
8. Create a secret using the cert with the following command:

{% code overflow="wrap" %}
```
kubectl create secret generic nOps-azuread --from-file myservice.cert --namespace nOps
```
{% endcode %}

9. With your existing Helm install command, append `-f values-saml.yaml` to the end.

At this point, test your SSO configuration to make sure it works before moving on to the next section. There is a Troubleshooting section at the end of this doc for help if you are experiencing problems.

## Entra ID RBAC configuration

### Admin/read only

The simplest form of RBAC in nOps is to have two groups: admin and read only. If your goal is to simply have these two groups, you do not need to configure filters. If you do not configure filters, this message in the logs is expected: `file corruption: '%!s(MISSING)'`

The [values-saml.yaml](https://github.com/nOps/poc-common-configurations/blob/main/saml-azuread/values-saml.yaml) file contains the `admin` and `readonly` groups in the RBAC section:

{% code overflow="wrap" %}
```
  rbac:
    enabled: true
    groups:
      - name: admin
        enabled: true # if admin is disabled, all SAML users will be able to make configuration changes to the nOps frontend
        assertionName: "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups" # a SAML Assertion, one of whose elements has a value that matches on of the values in assertionValues
        assertionValues:
          - "{group-object-id-1}"
          - "{group-object-id-2}"
      - name: readonly
        enabled: true # if readonly is disabled, all users authorized on SAML will default to readonly
        assertionName:  "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups"
        assertionvalues:
          - "{group-object-id-3}"
    customGroups: # not needed for simple admin/readonly RBAC
      - assertionName: "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups"
```
{% endcode %}

Remember the value of `assertionName` needs to match the claim name given in Step 2.5 above.

### Filtering

Filters are used to give visibility to a subset of objects in nOps. RBAC filtering is capable can filter for any types as the [Allocation API](/apis/monitoring-apis/api-allocation.md). Examples of the various filters available are these files:

* [filters.json](https://github.com/nOps/poc-common-configurations/blob/main/saml-azuread/filters.json)
* [filters-examples.json](https://github.com/nOps/poc-common-configurations/blob/main/saml-azuread/filters-examples.json)

These filters can be configured using groups or user attributes in your Entra ID directory. It is also possible to assign filters to specific users. The example below is using groups.

You can combine filtering with admin/read only rights, and it can be configured the same way. The same `assertionName` and values will be used, as is the case in this example.

The _values-saml.yaml_ file contains this `customGroups` section for filtering:

{% code overflow="wrap" %}
```
    customGroups: # not needed for simple admin/readonly RBAC
      - assertionName: "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups"
```
{% endcode %}

The array of groups obtained during the authentication request will be matched to the subject key in the _filters.yaml._ See this example _filters.json_ (linked above) to understand how your created groups will be formatted:

```
{
   "{group-object-id-a}":{
      "allocationFilters":[
         {
            "namespace":"*",
            "cluster":"*"
         }
      ]
   },
   "{group-object-id-b}":{
      "allocationFilters":[
         {
            "namespace":"",
            "cluster":"*"
         }
      ]
   },
   "{group-object-id-c}":{
      "allocationFilters":[
         {
            "namespace":"dev-*,nginx-ingress",
            "cluster":"*"
         }
      ]
   }
}
```

As an example, we will configure the following:

* Admins will have full access to the nOps UI and have visibility to all resources
* nOps users, by default, will not have visibility to any namespace and will be read only. If a group doesn't have access to any resources, the nOps UI may appear to be broken.
* The dev-namespaces group will have read only access to the nOps UI and only have visibility to namespaces that are prefixed with `dev-` or are exactly `nginx-ingress`

1. In the Entra ID left navigation, select _Groups_. Select _New group_ to create a new group.
2. For Group type, select _Security_. Enter a name your group. For this demonstration, create groups for `nOps_users`, `nOps_admin` and `nOps_dev-namespaces`. By selecting _No members selected_, Azure will pull up a list of all users in your organization for you to add (you can add or remove members after creating the group also). Add all users to the `nOps_users` group, and the appropriate users to each of the other groups for testing. nOps admins will be part of both the read only `nOps_users` and `nOps_admin` groups. nOps will assign the most rights/least restrictions when there are conflicts.
3. When you are done, select _Create_ at the bottom of the page. Repeat Steps 1-2 as needed for all groups.
4. Return to your created Enterprise application and select _Users and groups_ from the left navigation. Select _Add user/group_. Select and add all relevant groups you created. Then select _Assign_ at the bottom of the page to confirm.
5. Modify [filters.json](https://github.com/nOps/poc-common-configurations/blob/main/saml-azuread/filters.json) as depicted above.
   1. Replace `{group-object-id-a}` with the Object Id for `nOps_admin`
   2. Replace `{group-object-id-b}` with the Object Id for `nOps_users`
   3. Replace `{group-object-id-c}` with the Object Id for `nOps_dev-namespaces`
6.  Create the ConfigMap:

    ```
    kubectl create configmap group-filters --from-file filters.json -n nOps
    ```

You can modify the ConfigMap without restarting any pods.

{% code overflow="wrap" %}
```
kubectl delete configmap -n nOps group-filters && kubectl create configmap -n nOps group-filters --from-file filters.json
```
{% endcode %}

## Troubleshooting

You can look at the logs on the aggregator and cost-model containers. This script is currently a work in progress.

If `nOpsAggregator.enabled` is `true` or unspecified in _values.yaml_:

{% code overflow="wrap" %}
```
kubectl logs deployment/nOps-cost-analyzer -c cost-model --follow |grep -v -E 'resourceGroup|prometheus-server'|grep -i -E 'group|xmlname|saml|login|audience'
```
{% endcode %}

If `nOpsAggregator.enabled` is `false` in _values.yaml_:

{% code overflow="wrap" %}
```
kubectl logs services/nOps-aggregator --follow |grep -v -E 'resourceGroup|prometheus-server'|grep -i -E 'group|xmlname|saml|login|audience'
```
{% endcode %}

When the group has been matched, you will see:

```
2022-08-27T05:32:03.657455982Z INF AUDIENCE: [readonly group:readonly@nOps.com]
2022-08-27T05:51:02.681813711Z INF AUDIENCE: [admin group:admin@nOps.com]
```

{% code overflow="wrap" %}
```
configwatchers.go:69] ERROR UPDATING group-filters CONFIG: []map[string]string: ReadMapCB: expect }, but found l, error found in #10 byte of ...|el": "{ "label": "ap|..., bigger context ...|nFilters": [
         {
            "label": "{ "label": "app", "value": "nginx" }"
         }
     |...
```
{% endcode %}

This is what a normal output looks like:

{% code overflow="wrap" %}
```
2022-09-01T03:47:28.556977486Z INF   http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name: {XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:Attribute} FriendlyName: Name:http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name NameFormat: Values:[{XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:AttributeValue} Type: Value:admin@nOps.com}]}
2022-09-01T03:47:28.55700579Z INF   http://schemas.microsoft.com/identity/claims/tenantid: {XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:Attribute} FriendlyName: Name:http://schemas.microsoft.com/identity/claims/tenantid NameFormat: Values:[{XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:AttributeValue} Type: Value:<TENANT_ID_GUID>}}]}
2022-09-01T03:47:28.557019809Z INF   http://schemas.microsoft.com/identity/claims/objectidentifier: {XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:Attribute} FriendlyName: Name:http://schemas.microsoft.com/identity/claims/objectidentifier NameFormat: Values:[{XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:AttributeValue} Type: Value:<OBJECT_ID_GUID>}]}
2022-09-01T03:47:28.557052714Z INF   http://schemas.microsoft.com/ws/2008/06/identity/claims/groups: {XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:Attribute} FriendlyName: Name:http://schemas.microsoft.com/ws/2008/06/identity/claims/groups NameFormat: Values:[{XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:AttributeValue} Type: Value:<GROUP_ID_GUID>}]}
2022-09-01T03:47:28.557067146Z INF   http://schemas.microsoft.com/identity/claims/identityprovider: {XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:Attribute} FriendlyName: Name:http://schemas.microsoft.com/identity/claims/identityprovider NameFormat: Values:[{XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:AttributeValue} Type: Value:https://sts.windows.net/<TENANT_ID_GUID>/}]}
2022-09-01T03:47:28.557079034Z INF   http://schemas.microsoft.com/claims/authnmethodsreferences: {XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:Attribute} FriendlyName: Name:http://schemas.microsoft.com/claims/authnmethodsreferences NameFormat: Values:[{XMLName:{Space:urn:oasis:names:tc:SAML:2.0:assertion Local:AttributeValue} Type: Value:http://schemas.microsoft.com/ws/2008/06/identity/authenticationmethod/password}]}
2022-09-01T03:47:28.557118706Z INF Adding authorizations '[admin group:admin@nOps.com]' for user
2022-09-01T03:47:28.594663386Z INF Login called
2022-09-01T03:47:28.629402419Z INF Attempting to authenticate saml...
2022-09-01T03:47:28.629509235Z INF Authenticated saml
...
2022-09-01T03:47:29.11007143Z INF AUDIENCE: [admin group:seanp@teamnOps.onmicrosoft.com]
```
{% endcode %}
