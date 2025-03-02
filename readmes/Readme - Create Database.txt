# Create a database - Readme
# ==========================

# Description: This module is used to create database on File system/ASM. 
# Can be used to create cluster database.
# Multitenant database can also be created.
# Reference: https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/running-dbca-using-response-files.html#GUID-E84CE996-B30C-4DCA-AE4C-1E90201317C2

# Prerequisites:
# ==============
# Passwordless ssh needs to be setup between the Target lpar oracle owner and ansible controller user.

# Set the Variables for Oracle to execute this task: Open the file ansible_oracle_aix/roles/oradb-create/defaults/main.yml and modify the variables. Modify only the ones which are marked with comments.

  hostgroup: "{{ group_names[0] }}"
  oracle_dbca_rsp: "dbca_{{ item.0.oracle_db_name }}.rsp"        # Name of responsefile used by dbca. One per database
  oracle_netca_rsp: "netca_{{ item.home }}_{{ listener_name_template }}.rsp"
  oracle_user: oracle                        			# User that will own the Oracle Installations.
  oracle_user_home: "/home/{{ oracle_user }}" 			# Home directory for oracle_user. Needed for passing in ssh-keys, profiles etc
  oracle_group: oinstall                          		# Primary group for oracle_user.

  oracle_home_db: "{% if item.0 is defined %}{% if item.0.oracle_home is defined %}{{ item.0.oracle_home}}{% else %}{{ oracle_base }}/{{ item.0.oracle_version_db }}/{{ item.0.home }}{% endif %}{% else %}{% if item.oracle_home is defined %}{{ item.oracle_home }}{% else %}{{ oracle_base }}/{{ item.oracle_version_db }}/{{ item.home }}{% endif %}{% endif %}"

  oracle_stage: /u01/app/stage					# Location on the target AIX lpar to stage response files. Must be created manually.
  oracle_rsp_stage: "{{ oracle_stage }}/rsp"
  oracle_inventory_loc: /u01/app/oraInventory
  oracle_base: /u01/base
  oracle_profile_name: ".profile_{{ item.oracle_db_name }}"       # Name of profile-file. Sets up the environment for that database. One per database
  oracle_dbf_dir_fs: /oradata/db12c2                              # If storage_type=FS this is where the database is placed.
  oracle_reco_dir_fs: /oradata/db12c2                             # If storage_type=FS this is where the fast recovery area is placed.
  oracle_dbf_dir_asm: '+DATA2'                                    # If storage_type=ASM this is where the database is placed.
  oracle_reco_dir_asm: '+DATA2'                                   # If storage_type=ASM this is where the fast recovery area is placed
  datafile_dest: "{% if item.0.storage_type|upper == 'FS' %}{{ oracle_dbf_dir_fs }}{% elif item.0.storage_type|upper == 'ASM' %}{{ oracle_dbf_dir_asm }}{% else %}{% endif %}"
  recoveryfile_dest: "{% if item.0.storage_type|upper == 'FS' %}{{ oracle_reco_dir_fs }}{% elif item.0.storage_type|upper == 'ASM' %}{{ oracle_reco_dir_asm }}{% else %}{% endif %}"
  configure_cluster: True       				# Set it true when creating RAC database
  #oracle_install_option_gi: "none"
  oracle_gi_cluster_type: STANDARD
  hostgroup_hub: "{{ hostgroup }}-hub"
  hostgroup_leaf: "{{ hostgroup }}-leaf"
  create_listener: "{% if oracle_install_option_gi is defined %}False{% elif oracle_install_option_gi is undefined %}{% if item.listener_name is defined %}True{% else %}False{% endif %}{% endif %}"
  listener_name_template: "{% if item.listener_name is defined %}{{ item.listener_name }}{% else %}{{ listener_name }}{% endif %}"
  listener_protocols_template: "{% if item.listener_protocols is defined %}{{ item.listener_protocols }}{% else %}{{ listener_protocols }}{% endif %}"
  listener_port_template: "{% if item.listener_port is defined %}{{ item.listener_port }}{% else %}{{ listener_port }}{% endif %}"
  listener_name: ANSILIST
  listener_protocols: TCP
  listener_port: 1521
  autostartup_service: false

# Everything between the lines START-OF-PASSWORDS & END-OF-PASSWORDS can be
# put in an external passwords.yml file and be encrypted by Vault.
# The file should be put in 'group_vars/<your-config>/passwords.yml'
# This example will be broken out to a passwords.yml as soon as is allowed in ansible

## Start of passwords.

default_gipass: Oracle123				# asmsnmpPassword.

default_dbpass: Oracle123				# Common passwords for the database users. [Sys, system, pdbadmin, dbsnmp].

## End of passwords.

  dbca_templatename: General_Purpose.dbc
  dbca_initParams: "{% if '19.3' in item.0.oracle_version_db %} -initParams db_name={{item.0.oracle_db_name}}{% if item.0.oracle_db_unique_name is defined %},db_unique_name={{item.0.oracle_db_unique_name}}{% endif %}{% endif %}"

# This is an example layout of a database installation
  oracle_databases:                                            	# Dictionary describing the databases to be installed
        - home: db1                                            	# 'Last' directory in ORACLE_HOME path (e.g /u01/app/oracle/12.1.0.2/racdb)
          oracle_version_db: 19.3.0.0                    	# Oracle version
          oracle_home: /u01/app/19c_ansible			# Oracle Home location.
          oracle_edition: EE                                   	# The edition of database-server (EE,SE,SEONE)
          oracle_db_name: db19c                                 # Database name
          oracle_db_type: RAC                                   # Type of database (RAC,RACONENODE,SI)
          is_container: True                                  	# (true/false) Is the database a container database
          pdb_prefix: db19cpdb					# Pluggable database name.
          num_pdbs: 1						# Number of pluggable databases.
          storage_type: ASM                                     # Database storage to be used. ASM or FS.
          service_name: db19c                              	# Inital service to be created.
          oracle_init_params: ""                               	# Specific parameters to be set during installation. Comma-separated list
          oracle_db_mem_totalmb: 10000                          # Amount of RAM to be used for SGA + PGA
          oracle_database_type: MULTIPURPOSE                   	# MULTIPURPOSE|DATA_WAREHOUSING|OLTP
          redolog_size_in_mb: 50				# Redolog size in MB
          listener_name: LISTENER				# Listener name for creation
          state: present                                        # present | absent

# Executing the playbook: This playbook executes a role. Before running the playbook, open the playbook and update the hostname & remote user details as shown below. Do NOT change other parts of the script.
# Change directory to ansible_oracle_aix
# Name of the Playbook: create-db.yml
# Content of the playbook

- name: Create a Database
  hosts: rac91                  # Target Lpar hostname
  remote_user: oracle           # Remote username
  roles:
     - { role: oradb-create }

# ansible-playbook create-db.yml