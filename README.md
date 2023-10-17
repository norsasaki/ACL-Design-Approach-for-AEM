# ACL-Design-Approach-for-AEM

## Common mistakes in ACL design for AEM

1. each time a new request is made, the ACL/group design becomes more complex, leading to an increase in man-hours and defects
2. When multiple groups are combined, the number of man-hours required for testing increases enormously. Or, defects are discovered after operation due to inadequate testing.
3. GAPs are created between design and implementation due to lack of time to update design documents. No one will be able to grasp the correct ACL design.
4. There is no established method for releasing ACLs while maintaining integrity and power. Therefore, ACL settings differ slightly from environment to environment.
5. it is not known what kind of ACLs are necessary for each standard function of AEM to work properly. Therefore, it is discovered after operation that standard functions do not work.
6. Testing is performed manually. As a result, both completeness and test quality are not high.

## Why ACL design is difficult

* ACL design is difficult to begin with and requires knowledge and experience
* AEM provides only incomplete Role-based access control (RBAC)
  * Too few Built-in Groups. Too few Built-in Groups, too poor.
  * It's too strange to have to design from Read/Write/Create/Delete permission of Database layer to define a new Role for business users.
  * AEM's strength is its flexible content management. On the other hand, for authorization design, we need to decide the content design tightly. Because of the conflict, it is no wonder that AEM is not good at authorization management.
    * Is Attribute Based Access Control (ABAC) compatible?
* Because "which functions can be used" and "which contents can be accessed" are controlled by the same mechanism called ACL, a deep understanding of the authorization model is required when designing ACLs.
* ACLs cannot be managed by code.


## Best Practice for AEM ACL Design

1. use [Netcentric's AC Tool](https://github.com/Netcentric/accesscontroltool) 
2. Follow [Best Practices](https://github.com/Netcentric/accesscontroltool/blob/develop/docs/BestPractices.md). 
3. design the folder/node structure so that the ACL design is done at the same time as the folder/node design, and the ACL design is simple. 
4. Define and design requirements so that complex ACL design can be avoided. If complex ACLs are required, clearly explain their disadvantages and ask the client to carefully consider whether or not they are necessary.


## What's AC Tool

* The AC tool provide the simple way to manage the specification and deployment of complex Access Control Lists in AEM.

## Features of AC Tool

* easy-to-read Yaml configuration file format
* run mode support
* automatic installation with install hook
* cleans obsolete ACL entries when configuration is changed
* ACLs can be exported
* management of user's key stores and the global trust store
* stores history of changes
* ensured order of ACLs
* built-in expression language to reduce rule duplication
See [the slide of adaptTo() 2016](https://adapt.to/2016/presentations/adaptto2016-ac-tool-jochen-koschorke-roland-gruber.pdf) for details

## How can AC tool resolve the problems?

### How to avoid complications in ACL/group design?

It is most important to follow [Best Practices](https://github.com/Netcentric/accesscontroltool/blob/develop/docs/BestPractices.md). 
By doing so, you can create a flexible and prospective design.
The following is a list of some of the most important tips.

* Use fragment groups for functional aspects and content access
* Always use Allow statements. Avoid using a Deny statement
* Consider access rights when designing you content structure


### How can we prevent problems when combining multiple groups?

The problem is that if a `deny` rule is set for group A and an `allow` rule for group B, and a user belongs to both groups A and B, the ACLs will conflict and the expected behavior might not occur.

To avoid conflicts
* define `deny` rules only for the top level nodes
* Define only `allow` rules for lower level nodes, and don't use `deny` rules.


#### Example 

Consider the implementation of the following design

![](./img/acl-design-sample.drawio.svg)

* If ACLs are set normally, they will be inherited by child nodes.
* Therefore, to implement with only `allow`, we need a way to set ACLs that are not inherited by child nodes
* To set ACLs only on the parent node without allowing child nodes to inherit, use the following idiom


```
# The `deny` rules should be defined in fragment-restrict-for-everyone
- path: /content
  permission: deny
  actions: 
  privileges: jcr:all
  repGlob: 

- path: /content
  permission: allow
  actions: read,modify,create,delete
  privileges: 
  repGlob: ""  # matches node /foo only (no descendants, not even properties)

- path: /content
  permission: allow
  actions: read,modify,create,delete
  privileges: 
  repGlob: /jcr:*
```

* Sample Config for defining rules for green pages using this idiom
* See [here](https://github.com/norsasaki/ACL-Design-Approach-for-AEM/releases/tag/actool-example) for other configs
  * The actual project should be designed with a node hierarchy that is more easily implemented by ACLs, since users cannot add pages and it is not maintainable.

```
       - FOR path IN [/content/we-retail, /content/we-retail/A1]:
           - path: ${path}
             permission: allow
             actions: 
             privileges: jcr:read
             repGlob: ""
 
           - path: ${path}
             permission: allow
             actions: 
             privileges: jcr:read
             repGlob: /jcr:*
```

### Ideas to prevent obsolete design documents

The AC Tool allows you to define ACLs as YAML.
And since YAML can contain comment-outs, you can substitute an ACL design document with a YAML file by leaving comments for the ACL design.

![Alt text](./img/comment-in-yaml.png)



### How to release ACLs while maintaining integrity and power

* The AC Tool allows you to manage ACLs as code, and enables Git management and resource transfer just like Java code.
* It also provides integrity and idempotency, so ACLs can be released with confidence.

The ACL Packager tool from ACS Commons used to be the mainstream tool, but it did not have integrity and power, and when ACLs were transferred using ACL Packager, differences often appeared between environments unintentionally.



### Ideas for properly identifying ACLs required for standard functions.

* AEM provides several Built-in Groups, which can be extended or referenced depending on business requirements.
* The AC Tool can output ACLs for all groups in AEM in YAML format, so you can reduce omissions in ACL design by referring to the ACLs defined in the Built-in Groups.
  * For example, to publish a page, the `Replicate` authority of the template and policy is required in addition to the `Replicate` authority of the page itself.


### How to efficiently test ACLs

* Write test code and perform exhaustive ACL testing
  * CRUD operations can be done with `curl`, etc.
  * Manual testing should be performed as an aid.
* use [access-control-validator](https://github.com/Netcentric/access-control-validator)














## the Best Practice is great, but...

But, when actually designing according to Best Practice, the number of groups tends to be large and ACL design will be complex.
Therefore, we would like to introduce a more simpler design.

## I suggest that the more simplified design

![](./img/simplified-approach.drawio.svg)

### More simplified design #1

Best Practice suggests that Functional Fragment Groups should be created for each function, and only the necessary functions should be assigned to each role group.
However, it is difficult to design and implement this for each project, and although the AEM product should provide Functional Fragment Groups, this is not the case in reality.

Fortunately, the only roles required in many AEM projects are the following three types or their variants.
* Editor
* Publisher
* Approver
  
And these roles can be adequately represented by combining buit-in groups such as content-authors, dam-users, and workflow-users.
Therefore, if you do not create a Functional Fragment and use the built-in group as a substitute for the User Role fragment group, you can save cost and time in design.



### Useful Built-in Groups

The table below summarizes the reusable Built-in Groups.

| Built-in Groups         | Description                                                                                                                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| contributor             | Basic privileges that allow the user to write content (as in, functionality only).                                                                                                 |
| content-authors         | Group responsible for content editing. Requires read, modify, create, and delete permissions.                                                                                      |
| dam-users               | Out-of-the-box reference group for a typical AEM Assets user.                                                                                                                      |
| projects-administrators |                                                                                                                                                                                    |
| projects-users          |                                                                                                                                                                                    |
| user-administrators     | Authorizes user administration, that is, the right to create users and groups.                                                                                                     |
| workflow-administrators |                                                                                                                                                                                    |
| workflow-editors        | Group that is allowed to create and modify workflow models.                                                                                                                        |
| workflow-users          | A user participating in a workflow must be a member of group workflow-users. Gives the user full access to: /etc/workflow/instances so that they can update the workflow instance. |

Ref: [User Administration and Security](https://experienceleague.adobe.com/docs/experience-manager-65/administering/security/security.html?lang=en)


However, a built-in group may already have access rights to nodes to which it does not want to grant permissions.
For example, it is a very popular requirement that content-authors have read and write permissions under /content, but do not want to grant access rights except to specific sites under /content.
In such a case, you can reset the ACL of the Built-in Group by defining a rule to `deny` `/content` in `fragment-restrict-for-everyone`.

As shown in the figure below, the AC tool always adds a new rule to the bottom of the ACL. And when ACLs conflict, the rule located at the bottom of the list takes precedence.
By using this feature, ACLs defined in the Built-in Group can be reset like CSS reset.
Specifically, in the figure below, both `content-authors` and `fragment-restrict-for-everyone` have ACLs defined for `/content`.
Belonging to both `content-authors` and `fragment-restrict-for-everyone` will cause ACL conflicts, but the rule for `fragment-restrict-for-everyone` takes precedence because it is defined at a lower position in the ACL. The `fragment-restrict-for-everyone` rule has priority because it is defined lower in the ACL.



![](./img/priority-of-acl.png)

We know that we can reset the ACLs that the Built-in Groups have. So, how can we give arbitrary `allow` rules from that state?
This is very easy, just define the necessary `allow` rules in the AC Tools.

AC Tools will first register the `deny` rules defined in the YAML file, and then the `allow` rules.
As mentioned earlier, the `allow` rule takes precedence because the rule that is positioned at the bottom of the list takes precedence.


### Cases where Built-in Groups must be used

More simplified design #1 has another advantage.
In addition to ACLs, AEM may control authorization based on "whether or not a user belongs to a specific group".
For example, the following UI is displayed only if you belong to `workflow-users`.
Therefore, you need to belong to `workflow-users` to be able to use Language Copy correctly.


![](./img/sample-ui-managed-by-groups.png)


In light of this, it is more useful to establish a method of utilizing built-in groups rather than designing fragment groups.


```
# /libs/cq/translation/cloudservices/rendercondition/isWorkflowUser/isWorkflowUser.jsp
 if (checkUserGroup(resolver, userSession, workflow_users)) {
     return true;
 } 
```

Ref: [Render Condition](https://developer.adobe.com/experience-manager/reference-materials/6-5/granite-ui/api/jcr_root/libs/granite/ui/docs/server/rendercondition.html#)

## More simplified design #2

Best Practice says that read/write access to contents should be provided by content groups.
While we agree in principle, we believe there is a more practical approach in the following respects.

* Business requirements determine which access rights are granted to content. However, since the required permissions vary in detail depending on roles and content, there is concern that creating fragment groups in terms of Read/Write will result in a very large number of groups.
* Whether content can be published or not is intuitively a functional perspective, but inside AEM it is controlled as one of the ACLs. Therefore, it is better to manage the publishing authority itself in the content group, not in the fragment group, for better visibility.

Therefore, it is better to create a content group that corresponds to a role on a one-to-one basis, and to manage all ACLs for the contents necessary for that role in that group.
It is also better to manage publishing privileges in the content group.




## Case Study | Requirements

Consider ACL and group design for the following requirements.

* Multi-site (We-Retail)
* Head office manages language-master content and deploys to each country/language using Live Copy and Language Copy.
* Branch offices manage the contents of their own region sites.
* Approver's product is always required to publish a page
* Page editors can create and update pages
* Page publishers can create, update, delete, and publish pages

## Case Study | Global Fragments

Global Fragment defines the following two groups

* fragment-restrict-for-everyone
  + Define only `Deny` rule.
  + Only `Deny` rules are defined in this group.
  + Define ACLs to deny access to the following user content areas.
     
    - /content/dam/projects, /content/dam/collections, /content/cq:tags, /conf
  + ACLs defined in the Built-in Group that need to be overridden are also defined here
* fragment-basic-allow
  + Define `Allow` rules for all sites and groups.
  + Mainly, rules for granting access to parent nodes, but not to child nodes, and only to parent nodes.
    - [White listing nodes](https://github.com/Netcentric/accesscontroltool/blob/develop/docs/BestPractices.md#white-listing-nodes)


## Case Study | Content Fragments

Content Fragments should be created according to the following policy

* Separate Content groups for each region, since language-masters and administrators of each region site are different.
* Create separate roles for editors, publishers, and approvers for each region.
  + `content-${sitePrefix}-${country}-for-${role}`
* ACLs to be assigned to each role are as follows
     
    | role      | priviledge                         |
    | --------- | ---------------------------------- |
    | editor    | read,modify,create                 |
    | publisher | read,modify,create,delete,acl_read |
    | approver  | read,modify,create,delete,acl_read |

     

* See [here](https://github.com/norsasaki/ACL-Design-Approach-for-AEM/releases/tag/actool-example) for a sample configuration of a case study

