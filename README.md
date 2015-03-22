# Make-wait_condition-work-on-openstack-heat-without-admin-role
When you with heat on openstack (icehouse,juno,kilo), and you need to use wait_conditions, you are quicky confronted with the flowing problem :
Resource CREATE failed: Forbidden: You are not authorized to perform the requested action:identity:list_roles (Disable debug mode to suppress these details.) (HTTP 403)   
here is the result of 
```
heat stack-show stack-simple6
+----------------------+---------------------------------------------------------------------------------------------------------------------+
| Property             | Value                                                                                                               |
+----------------------+---------------------------------------------------------------------------------------------------------------------+
| capabilities         | []                                                                                                                  |
| creation_time        | 2015-03-22T14:15:41Z                                                                                                |
| description          | Heat demo template for wait conditions                                                                              |
| disable_rollback     | True                                                                                                                |
| id                   | 8aeaefa7-06df-4681-a441-e0dd4cb4274d                                                                                |
| links                | http://xxx:8004/v1/515ba05e6aa943f194e39e52654c79a1/stacks/stack-simple6/8aeaefa7-06df-4681-a441-e0dd4cb4274d |
| notification_topics  | []                                                                                                                  |
| outputs              | []                                                                                                                  |
| parameters           | {                                                                                                                   |
|                      |   "instance_type": "m1.extra_tiny",                                                                                 |
|                      |   "OS::stack_id": "8aeaefa7-06df-4681-a441-e0dd4cb4274d",                                                           |
|                      |   "OS::stack_name": "stack-simple6"                                                                                 |
|                      | }                                                                                                                   |
| parent               | None                                                                                                                |
| stack_name           | stack-simple6                                                                                                       |
| stack_owner          | demo                                                                                                            |
| stack_status         | CREATE_FAILED                                                                                                       |
| stack_status_reason  | Resource CREATE failed: Forbidden: You are not                                                                      |
|                      | authorized to perform the requested action:                                                                         |
|                      | identity:list_roles (Disable debug mode to suppress                                                                 |
|                      | these details.) (HTTP 403)                                                                                          |
| template_description | Heat demo template for wait conditions                                                                              |
| timeout_mins         | None                                                                                                                |
| updated_time         | None                                                                                                                |
+----------------------+---------------------------------------------------------------------------------------------------------------------+

```


So you have the tentation to add admin role to the user on his tenant but it does not help you since the admin role gives extended rights to the user on all the openstack cloud.

A detailed explanation of the problem and it is origin is given here

http://hardysteven.blogspot.fr/2014/04/heat-auth-model-updates-part-1-trusts.html
http://hardysteven.blogspot.fr/2014/04/heat-auth-model-updates-part-2-stack.html

Steve hardy gives a well explained details on heat auth model but i could not figure out how get wait_condition work on my cloud.

After a weekend of work and test, I can give a detailed procedure for that 

To make wait_condition work on openstack heat without requiring admin role, you have to :

1- Install python-openstackclient in order to use keystone v3 api calls 
2- Create a domain for users and projects create by heat  (replace <token> by  token admin )

```
openstack --os-token <token> --os-url=http://identity:5000/v3 --os-identity-api-version=3 domain create heat --description 'Owns users and projects created by heat'
```

3- Create a user on heat domain. This user will manage users and projects created by heat (replace <token> by  token admin and <heat_domain_id> with the heat domain id)

```
openstack --os-token  <token>  --os-url=http://identity:5000/v3 --os-identity-api-version=3 user create --password password --domain <heat_domain_id> heat_domain_admin --description 'Manages users and projects created by heat'

```

4- Give the user create later admin role on the heat domain
```
openstack --os-token <token>  --os-url=http://identity:5000/v3 --os-identity-api-version=3 role add --user heat_domain_admin --domain <heat_domain_id>  admin
```

5- Edit /etc/heat/heat.conf and change theses values :
```
deferred_auth_method=trusts
trusts_delegated_roles=heat_stack_owner
stack_user_domain_id=<heat_domain_id> (replace <heat_domain_id> with the heat domain id)
stack_domain_admin=heat_domain_admin
stack_domain_admin_password=password
```

6- Restart Heat services : for i in {api,api-cfn,engine}; do service heat-$i restart;done

7- Make sure you have these roles for the user executing the stack : Member,_member_,heat_stack_user,heat_stack_owner

Now you can use wait_condition, without admin role.




