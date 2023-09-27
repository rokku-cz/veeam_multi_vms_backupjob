# Ansible Collection - rokku.veeam_multi_vms_backupjob

Documentation for the collection.

# Custom veeam job for creation of multi VM's backup jobs
This module was adjusted in order to create multi VM's backup jobs via Veeam REST API. 

**Module input parameters:** 
server_username=dict(type='str', required=True),
server_password=dict(type='str', required=True, no_log=True),
server_port=dict(type='str', default='9419'),
state=dict(type="str", choices=("absent", "present"), default="present"),
id=dict(type='str', required=False),
jobName=dict(type='str', required=False),
jsonDefinition=dict(type='str', required=True),
type=dict(type='str', choices=("VirtualMachine", "vCenterServer"), default="VirtualMachine"),
backupRepositoryId=dict(type='str', required=False),
description=dict(type='str', required=False),
daily_backup_enable=dict(type='bool', required=False),
daily_backup_localtime=dict(type='str', required=False),
validate_certs=dict(type='bool', default='false')

Based on provided inputs parameters the body payload is composed. 

# Usage example via ansible playbook veeam_create_job_multi.yml

- name: end-to-end create veeam job
  hosts: localhost
  gather_facts: false
  vars:
    repo_name: 'Backup Repository 1'
    vcenter_hostname: <VCENTRE_HOSTNAME/IP> 
    vcenter_username: <VCENTRE_USERNAME>
    vcenter_password: <VCENTRE_PASSWORD>
    veeam_server_name: <VEEAM_HOSTNAME/IP>
    veeam_server_username: <VEEAM_USERNAME>
    veeam_server_password: <VEEAM_PASSWORD>
    veeam_backup_job_name: <VEEAM_JOB_NAME>
    veeam_backup_job_description: <VEEAM_JOB_DESCRIPTION>
    veeam_daily_backup_auto_enable: <VEEAM_TRUE/FALSE>
    veeam_daily_backup_localtime: <VEEAM_TIME"
    veeam_servers_list:
      - splunk.local
      - splunk.local_restored
      - zabbix.local

  tasks:
    - name: Get VBR Repos
      veeamhub.veeam.veeam_vbr_rest_repositories_info:
        server_name: '{{ veeam_server_name }}'
        server_username: '{{ veeam_server_username }}'
        server_password: '{{ veeam_server_password }}'
      register: repo_testout

    - name: Debug VBR Repos Result
      ansible.builtin.debug:
        var: repo_testout
    - name: Filter Repo Object
      set_fact:
        repo_id: "{{ repo_testout | json_query(repos_id_query) }}"
      vars:
        repos_id_query: "infrastructure_repositories.data[?name==`{{ repo_name }}`].id"

    - name: Gather all registered virtual machines
      vars:
      community.vmware.vmware_vm_info:
        hostname: '{{ vcenter_hostnameAnsible Test description" }}'
        username: '{{ vcenter_username }}'
        password: '{{ vcenter_password }}'
        vm_name: '{{ item }}'
        validate_certs: no
      delegate_to: localhost
      register: vminfo
      loop: '{{ veeam_servers_list }}'

    - set_fact:
        final_list: "{{ final_list | default([]) + [{ 'hostName' : vcenter_hostname, 'name' : item.name, 'objectId' : item.objectId, 'type' : 'VirtualMachine'}] }}"
      loop: "{{ vminfo.results | json_query('[].virtual_machines[].{hostName:esxi_hostname, name: guest_name, objectId: moid }') }}"

    - debug:
        msg: "{{ final_list }}"

    - name: Create VBR Job
      veeamhub.veeam.veeam_vbr_rest_jobs_manage_custom:
        server_name: '{{ veeam_server_name }}'
        server_username: '{{ veeam_server_username }}'
        server_password: '{{ veeam_server_password }}' 

        state: present
        jobName: '{{ veeam_backup_job_name }}' 
        description: '{{ veeam_backup_job_description }}'
        daily_backup_enable: '{{ veeam_daily_backup_auto_enable }}'
        daily_backup_localtime: '{{veeam_daily_backup_localtime }}'
        jsonDefinition: "{{ final_list }}"

        #description: "Created by Ansible RestAPI Module"
        backupRepositoryId: "{{ repo_id[0] }}"
      register: create_job
    - name: Debug VBR Jobs Result
      ansible.builtin.debug:
        var: create_job


