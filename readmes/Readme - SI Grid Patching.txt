# Patch Grid Infrastructure 
# =========================

# Description: This module is used to patch Grid Infrastructure, both standalone & RAC.
# This is exclusive for applying Release update patches for 19c.
# It does "prereq CheckConflictAgainstOHWithDetail" before applying the patch and exits if there are any conflicts.
# This Readme contain Two sections. 1) Patch Standalone GI, 2) Patch RAC GI.

# Prerequisites:
# ==============

# Passwordless ssh needs to be setup between the Grid owner and ansible controller user.
# The Grid owner should have sudo access to apply patch using opatchauto.

##########################
# 1. Patch Standalone GI #
##########################

# Set the Variables to execute the task: Open the file ansible_oracle_aix/roles/oraswgi-manage-patches/defaults/main.yml and modify the variables. Modify only the ones which are marked with comments.

hostgroup: "{{ group_names[0] }}"
cluster_master: "{{play_hosts[0]}}"
configure_cluster: false                # Set this to True in case of Grid Infrstructure RAC. Set it to False in case of Standalone Grid Infrstructure (Oracle Restart).
grid_install_user: oracle               # Grid installation home Owner.
oracle_group: oinstall                          # Primary group for oracle/grid user.
apply_patches_gi: True
patch_before_rootsh: True

oracle_sw_source_local: /home/ansible/ansible_test/patches      # This is the location on Ansible controller, place the appropriate OPatch & patch zip files.
is_sw_source_local: true                                        # Set this to True to utilize the patches mentioned in the above parameter oracle_sw_source_local. "install_from_nfs" must be set to False when this parameter is set to True.
install_from_nfs: false                                         # Set this parameter to True if you want to use NFS mount point to stage the patches. Associated parameter is "nfs_server_sw_path".
oracle_sw_copy: "{% if install_from_nfs %}False{% else %}True{% endif %}"
oracle_sw_unpack: "{% if install_from_nfs %}False{% else %}True{% endif %}"
nfs_server_sw: nfsserver
nfs_server_sw_path: /path/to/orasw                              # Location of the NFS path mounted on AIX lpar where the patches and OPatch must be placed. No need to extract the zip files, ansible will do it. Make sure this directory has proper read permissions.
oracle_stage: /u01/app/patch_stage                                  # Location on the AIX Lpar where Ansible will stage the patch files. Must be created manually.
oracle_stage_remote: "{{ oracle_stage }}"
oracle_stage_install: "{% if not oracle_sw_copy and not oracle_sw_unpack%}{{ oracle_stage_remote }}{% else %}{{ oracle_stage }}{% endif %}"
oracle_rsp_stage: "{{ oracle_stage }}/rsp"
oracle_patch_stage: "{{ oracle_stage }}/patches"
oracle_patch_stage_remote: "{{ oracle_stage_remote }}/patches"
oracle_patch_install: "{% if not oracle_sw_copy and not oracle_sw_unpack%}{{ oracle_patch_stage_remote }}{% else %}{{ oracle_patch_stage }}{% endif %}"

oracle_sw_patches:
   - filename: RU19.14-p33509923_190000_AIX64-5L.zip            # Patch filename placed on the Ansible controller (defined in oracle_sw_source_local (or) nfs_server_sw_path).
     patchid: 33509923                                          # Patch ID
     version: 19.3.0.0                                          # Oracle version.
     patchversion: 19.9.0.0.201020                              # Release update patch version (or) Patch ID
     description: GI-RU-Oct-2020                                # This is just a description which is optional.

oracle_opatch_patch:
   - filename: p6880880_12201029_AIX64.zip                      # OPatch zip filename.
     version: 19.3.0.0                                          # Database Version

oracle_base: /u01/app/oracle
oracle_install_version_gi: 19.3.0.0                   # Grid Infrastructure version. Example: 19.3.0.0 or 12.2.0.1. Make sure to follow the same pattern throughout this file. Example: For version: 19.3, if 19.3.0.0 has to be used throughtout this file.
oracle_inventory_loc: /u01/app/oraInventory             # Oracle Inventory Location.
oracle_hostname: "{{ ansible_fqdn }}"                            # Full (FQDN) name of the host
ocm_response_file: "{{ oracle_patch_stage }}/{{ oracle_install_version_gi }}/ocm.rsp"

gi_patches:
     oracle_install_version_gi: 19.3.0.0              # Grid Infrastructure version.
     19.3.0.0:
          opatch_minversion: 12.2.0.1.29              # Minimum opatch version required to apply the patch as per Oracle readme.txt file
          opatchauto:
              - patchid: 33509923                     # Release update Patch ID to be applied.
                patchversion: 19.14.0.0.220118        # Release update patch version obtained from Oracle readme.txt file
                state: present                        # present - applies this patch, absent - rollbacks the patch.
                subpatches:
                    - 33575402                        # subpatches list inside the Release update patch directory.
          opatch: []
#If the version of Grid infrastructure is 12.2+ use the following parameters.
     12.2.0.1:
          opatch_minversion: 12.2.0.1.12
          opatchauto:
              - patchid: 28183653
                patchversion: 12.2.0.1.180717
                state: absent
                subpatches:
                    - 28163133
                    - 28163190
                    - 28163235
                    - 26839277
                    - 27144050

oracle_home_gi: "{% if configure_cluster %}{{ oracle_home_gi_cl }}{% else %}{{ oracle_home_gi_so }}{% endif %}"
oracle_home_gi_cl: "/u01/app/19c/grid" # ORACLE_HOME for Grid Infrastructure (Clustered)
oracle_home_gi_so: "/u01/grid19c" # ORACLE_HOME for Grid Infrastructure (Stand Alone)

# Executing the playbook: This playbook executes a role. Before running the playbook, open the playbook and update the hostname & remote user details as below. Do NOT change other parts of the script.
# Change directory to ansible_oracle_aix
# Name of the Playbook: gi-si-opatchauto.yml
# Content of the playbook

- name: Apply patches to GI
  hosts: ansible_db                     # Hostname of the target AIX lpar.
  remote_user: oracle                   # Grid Home Owner.
  become: yes                           # For root access.
  roles:
     - {role: oraswgi-manage-patches }

# ansible-playbook gi-si-opatchauto.yml --ask-become-pass

###################
# 2. Patch RAC GI #
###################

# Set the Variables to execute the task: Open the file ansible_oracle_aix/roles/oraswgi-manage-patches/defaults/main.yml and modify the variables. Modify only the ones which are marked with comments.
# All the variables are same as in above section except the following.

configure_cluster: True                # Set this to True in case of Grid Infrstructure RAC. Set it to False in case of Standalone Grid Infrstructure (Oracle Restart).

# Executing the playbook: This playbook executes a role. Before running the playbook, open the playbook and update the hostname & remote user details as below. Do NOT change other parts of the script.
# Change directory to ansible_oracle_aix
# Name of the Playbook: gi-rac-opatchauto.yml
# Content of the playbook

 - name: Apply patches to GI
   hosts: rachosts      # AIX Lapr hostgroup name defined in Ansible inventory.
   become: yes
   remote_user: grid    # Remote AIX Lpar username.
   serial: 1
   roles:
      - {role: oraswgi-manage-patches }
      
# ansible-playbook gi-rac-opatchauto.yml --ask-become-pass