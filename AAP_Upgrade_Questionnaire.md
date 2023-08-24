# AAP Questionnaire

## 1.	Do you have your previous inventory file and/or credentials file if kept separate?
*	if yes please provide this to us.
*	If no provide the following information
    *	All hostnames and ips, and role
    *	Database names and credentials


## 2.	Do you have registry.redhat.io credentials ready?
*	if "no" please create a service account credential following this guide from red hat and provide that to us https://access.redhat.com/RegistryAuthentication
*	if yes please provide that to us  
    * Note the service account token/pw will be very long


## 3.	Do you have load balancing urls?
*	This helps offload  SSL certificates management and allows for side by side upgrade
*	URLs needed can include:
    *	Ansible Controller node (tower)
    *	Ansible hub node ( collections proxy, container registry)
    *	Event Driven Ansible Controller (new in 2.4)


## 4.	Do you wish to have LDAP / AD Authentication for automation hub? 
* https://access.redhat.com/articles/6977659 (hub ldap consolidation)
    *	We recommend creating a new ansible hub users group 
        * separate from controller groups *not everyone will need to use / access hub*
    *	*Required to be applied during installation*
    ```
    automationhub_authentication_backend = "ldap"
    automationhub_ldap_server_uri = "ldap://ldap:389"   (for LDAPs use  automationhub_ldap_server_uri = "ldaps://ldap-server-fqdn")
    automationhub_ldap_bind_dn = "cn=admin,dc=ansible,dc=com"
    automationhub_ldap_bind_password = "GoodNewsEveryone"
    automationhub_ldap_user_search_base_dn = "ou=people,dc=ansible,dc=com"
    automationhub_ldap_group_search_base_dn = "ou=people,dc=ansible,dc=com"
    ```
    *	Optional filters and parameters
    ```
    auth_ldap_user_search_scope: "{{ automationhub_ldap_user_search_scope | default('SUBTREE') }}"
    auth_ldap_user_search_filter: "{{ automationhub_ldap_user_search_filter | default('(uid=%(user)s)') }}"
    auth_ldap_group_search_scope: "{{ automationhub_ldap_group_search_scope | default('SUBTREE') }}"
    auth_ldap_group_search_filter: "{{ automationhub_ldap_group_search_filter | default('(objectClass=Group)') }}"
    auth_ldap_group_type_class: "{{ automationhub_ldap_group_type_class | default('django_auth_ldap.config:GroupOfNamesType') }}"

    ```
    *	Any extra LDAP parameters that need to be set must be defined in a dictionary named ldap_extra_settings, for example one can create a YAML file as such:
    ```
    #ldapextras.yml   
    ldap_extra_settings:
    AUTH_LDAP_USER_ATTR_MAP: '{"first_name": "givenName", "last_name": "sn", "email": "mail"}'
    ```

## 5.	 Do you wish to use Trusted SSL Certificates?
* If yes these will need provided to us
*	Our recommendation is to create a single cert for all systems in the san
    *	IP address, shortname, and fqdn of all devices
        * see upgrade document for additional information

## 6.	Setup Event-Driven Ansible Controller?
*	If yes we need the following or we can create and pass along
    ```
    automationedacontroller_admin_password=''
    automationedacontroller_pg_password=''
    ```
