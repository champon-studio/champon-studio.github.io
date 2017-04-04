---

title: "RabbitMQ + LDAP"
post_author: ferrisky
tags: rabbitmq
 
---

RabbitMQ was used in lots of systems to act as message exchange platform/bus. For real world applications, based on different requirements of solution, the RabbitMQ will be configured with private account/password. This is to prevent connections from unauthenticated user.

In RabbitMQ, there is an built-in database for account management, it is good but may not enough for applications that have requirement to integrate account from external systems. RabbitMQ provides some solutions if teams want to using external sources for Authentication & Authorization (A&A). The one introduced here is using LDAP.

LDAP is common in use of account/resource managements, all identified item(both human and non-human) having a Distinguished Name (DN), such as `CN=FerryChen,OU=IT,DC=abc,DC=com`; or `CN=OfficePrinter,OU=Sales,DC=dtc,DC=com`. The DN is used to point to an uniqe object, and all object can have password or credentials for A&A process.

RabbitMQ provides plugins to configure RabbitMQ using external LDAP for its account database for A&A. To make it works, there are two parts have to be taken.

1. Configuration for Authentication
2. Configuration for Authorization

---

### Configuration for Authentication

- Enable RabbitMQ LDAP plugin

```
rabbitmq-plugins enable rabbitmq_auth_backend_ldap
```

- Set LDAP plugin as backend for A&A used by RabbitMQ

```
{rabbit,[{auth_backends, [rabbit_auth_backend_ldap]}]}
```

>*[optional]* RabbitMQ can be configured with multiple A&A backend, the order of backend listed will be the priority of backends. For example, using LDAP first, and then try internal one if LDAP having no such account.

>```
{rabbit,[{auth_backends, [rabbit_auth_backend_ldap, rabbit_auth_backend_internal]}]}
```

> *[optional]* RabbitMQ can also be configured with multiple A&A backend, and asked to pass LDAP in the second attempt when LDAP not contains the account

>```
{rabbit,[{auth_backends, [{rabbit_auth_backend_ldap, rabbit_auth_backend_internal},rabbit_auth_backend_internal]}]}
```

- Configure RabbitMQ to fill the template of DN from login accounts. Because LDAP items are identified by DN, RabbitMQ using `user_dn_pattern` to define how login account be mapped to LDAP DN.
`$username` is the login account, which will automatically filled by RabbitMQ.

```
{user_dn_pattern, "cn=${username},ou=IT,dc=ABC,dc=com"}
``` 

> *[advanced]* RabbitMQ not only support `user_dn_pattern` which may not flexiable enough for some use-case, but also allowed to be configured to search LDAP DN base with specified attributes. Please refer to [official website](https://www.rabbitmq.com/ldap.html)

- The rest parts are easy, just to configure RabbitMQ with access point of LDAP server.


```
{rabbitmq_auth_backend_ldap,
   [ {servers,               ["my-ldap-server"]},
     {use_ssl,               false},
     {port,                  389},
     ...
   ]
 }
```

---

### Configuration for Authorization

There are three iteam type of authorization introduced by RabbitMQ LDAP plugin: 1) vhost, 2) resource, and 3) tag. All of authorization are based on `query` in form `{type , <query>}`. If `<query>` returns `true`, the item type will be authorized to have access rights.

- To constraint the vhost, e.g. only allow users in specific group can access the vhost.

```
{vhost_access_query, {in_group, "cn=${vhost}-users,ou=vhosts,dc=example,dc=com"}}
```

Similarly, users only allowed to access vhost with its name also be possible. `${vhost}` is a predefined variable visit [here](https://www.rabbitmq.com/ldap.html) for more info.

```
{vhost_access_query, {exists, "ou=${vhost},ou=vhosts,dc=example,dc=com"}}
```

- Set up authorizaiton rules for resources. There are two type of resource in RabbitMQ, exchange and queue, RabbitMQ LDAP plugin allows user define access rule of these two resources separately. Futhermore, each resource type has configure,read, and write permission, which can also be defined. As the result, there are six combinations on types of resources and permissions for authorization. Listing below is an example of how to define authorization on resource accesses.

```
{resource_access_query,
 {for, [{resource, exchange, {for, [{permission, configure,
                                     {in_group, "cn=wheel,dc=example,dc=com"}
                                    },
                                    {permission, write, {constant, true}},
                                    {permission, read,  {constant, true}}
                                   ]}},
        {resource, queue,    {constant, true}}]}}
```

The example above displays that only users defined in LDAP group cn=wheel,dc=example,dc=com can create/destory exchanges, but everyone are able to read/write/attach to existed exchanges.

- Tag is used to decide the role of logined user.checkout [here](https://www.rabbitmq.com/management.html) to see the difference of roles. listing below define that all users are roles with management, but no one can be administrator.

```
{tag_queries,           [{administrator, {constant, false}},
                         {management,    {constant, true}}]}
```
> Folloing example reveals that conditions can be very flexible. It support and,or, and not logic operators as well as regular expressions.
> 
> ```
>   {resource_access_query,
       {'or', [
               {'and',[{equals,{attribute, "${user_dn}", "cn"},"onboard"} ,{match, {string, "${name}"}, "^\\$gaia\/[A-Za-z0-9_-]+\/[A-Za-z0-9_-]+\/v[0-9]+\/onboard\/[0-9A-Fa-f]{12}\/(req|rsp|\\+)\/(\\+|[A-Za-z0-9_-])+"}]},
               {'and',[{'not', {equals, {attribute, "${user_dn}", "cn"}, "onboard"}}, {match, {string, "${name}"},{string, "^\\$gaia\/[A-Za-z0-9_-]+\/[A-Za-z0-9_-]+\/[A-Za-z0-9_-]+\/[A-Za-z0-9_-]+\/(([0-9]{1,3})-([0-9]{1,3})-([0-9]{1,3})-([0-9]{1,3})\/)?(req|rsp|\\+)\/(\\+|[A-Za-z0-9_-])+"}}, {match, {attribute, "${user_dn}", "cn"}, {string, "${name}"}}]},
               {equals, {attribute, "${user_dn}", "cn"},"gaia_dl"},
               {equals, {string, "${name}"},"amq.topic"},
               {'and', [{match, {string, "${name}"},"^mqtt-subscription-[A-Za-z0-9_-]+qos[0-2]$"},{match, {string, "${name}"},{attribute, "${user_dn}", "cn"}}]}
              ]
       }
    }
```