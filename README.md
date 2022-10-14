# Ansible EIGRP and redistribution example using Jinja2 templates

#### Short summary
Eve-ng topology is stored in **eveng/** directory. I used eve-ng pro version. Please use Python virutal Environments.This was compiled to support ansible 2.9.4 version howerver it also should work with release >= 2.10.x 
Backups are stored in backups/ dir.

#### Please use Python venv
`source venv/bin/activate`

#### How to push configuration with Ansible playbook(s). 


It requreis to execute only a single command:

    `ansible-playbook eigrp_config.yml`

For this lab I use jinja2 template to configure interfaces as well as EIGRP settings and redistribution.

##### Device variables.

```yaml
interfaces:
 - name: Ethernet0/0
   description: Link to R2 configured by Ansible
   ipv4: 10.1.12.1/24
   ipv6: 2001:db8:acad:12::1/64
   link_local: fe80::12:1

 - name: Ethernet0/1
   description: Link to D1 configured by Ansible
   ipv4: 10.1.11.1/24
   ipv6: 2001:db8:acad:11::1/64
   link_local: fe80::11:1
   AS: "64512"
   cdp: 'no'

 - name: Loopback 1
   description: Loopback 1 configured by Ansible
   ipv4: 10.1.1.1/24
   ipv6: 2001:db8:acad:1::1/64
   link_local: fe80::1:1

eigrp:
  instances:
    - asn: '64512'
      rid: '1.1.1.1' 
      mode: 'classic'
      redistribute: '64513'
      networks:
        - 10.1.11.0/24
    - asn : '64513'
      mode: 'named'
      rid: '1.1.1.1' 
      asname: 'AS_CISCO'
      redistribute: '64512'
      networks:
        - 10.1.1.0/24
        - 10.1.12.0/24
      #redistribute: static
      #stub: static

```


##### Jinja2 template for interface configuration

```jinja2
#jinja2: lstrip_blocks:True, trim_blocks:True
{% for intf in interfaces %}
interface {{ intf.name }}
{% if intf.description is defined %}
   description {{ intf.description }} 
{% endif %}
{% if intf.mode is defined and intf.mode =="L3" %}
   no switchport
{% endif %}
{% if intf.ipv4 is defined  %}
   ip address {{ intf.ipv4 | ipaddr('address') }} {{ intf.ipv4 | ipaddr('netmask')}}
{% endif %}
{% if intf.ipv6 is defined %}
   ipv6 address {{ intf.ipv6 }}
{% endif %}
{% if intf.link_local is defined %}
   ipv6 address {{ intf.link_local}} link-local
{% endif %}
{% if intf.cdp is defined and intf.cdp =='no' %}
   no cdp enable
{% endif %}
   no shutdown
{% endfor %}
```
#### Main taks to configure router interfaces.
``` yaml
---
# Tasks file for rtr_interfaces

- name: " Build IOS interface config "
  ansible.builtin.template:
    src: "interfaces.j2"
    dest: "config/{{ inventory_hostname }}_interface_config_lstrip.txt"

- name: " Send interface interface basic config to device(s)"
  cisco.ios.ios_config:
    src: "config/{{ inventory_hostname }}_interface_config_lstrip.txt"
    backup: "yes"
    save_when: "modified"
```

#### Template for EIGRP settings.

``` jinja2
{% for instance in eigrp.instances -%}
{% if instance.mode == 'classic' -%}
router eigrp {{ instance.asn }}
   {%+ if instance.rid is defined -%} 
    eigrp router-id {{ instance.rid }}
   {% endif -%}
  {% for network in instance.networks %}
  network {{ network | ipaddr('network') }} {{ network | ipaddr('wildcard') }}
  {% endfor %}
  {% if instance.stub is defined %}
   stub {{ instance.stub }}
  {% endif %}
exit
{#{% endif -%}
{%- if  instance.mode == 'named'  -%}#}
{% else -%} {# named mode #}
router eigrp {{ instance.asname }}
  address-family ipv4 unicast autonomous-system {{ instance.asn }}
   {%+ if instance.rid is defined -%} 
    eigrp router-id {{ instance.rid }}
   {% endif -%}
  {% for network in instance.networks %}
  network {{ network | ipaddr('network') }} {{ network | ipaddr('wildcard') }}
  {% endfor %}
  {% if instance.stub is defined -%} 
   stub {{ instance.stub }}
  {% endif %}
exit-address-family
 {% endif -%}
{% endfor -%}
```
