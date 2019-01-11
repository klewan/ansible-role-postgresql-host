Ansible Role: postgresql-host
=============================

This role configures a Linux server for PostgreSQL Database.

Supported OS:
-------------
* RedHat
* CentOS
* OracleLinux

Requirements
------------

None

Role Variables
--------------

Available variables are listed below, along with default values (see `defaults/main.yml`):

    # min supported OS and version
    postgresql_host_os_family_supported: "RedHat"
    postgresql_host_os_min_supported_version: "7.0"

    # commands used to disable transparent hugepage in runtime
    postgresql_host_transparent_hugepage_disable:
      - {disable: "echo never >", path: /sys/kernel/mm/transparent_hugepage/enabled, rclocal: /etc/rc.d/rc.local}
      - {disable: "echo never >", path: /sys/kernel/mm/transparent_hugepage/defrag, rclocal: /etc/rc.d/rc.local}

    # disable/enable selinux
    postgresql_host_disable_selinux: true

    # firewall service name
    postgresql_host_firewall_service: "{% if ansible_distribution_major_version|int==6%}iptables{%elif ansible_distribution_major_version|int==7 %}firewalld{% else %}0{% endif %}"

    # disable/enable firewall service
    postgresql_host_disable_firewall: false

    # custom firewalld config
    postgresql_host_enable_custom_firewalld_config: true
    postgresql_host_custom_firewalld_config: []
    #  - { port: 5432/tcp, permanent: true, immediate: true, state: enabled, comment: 'PostgreSQL connection' }
    #  - { port: 5432/udp, permanent: true, immediate: true, state: enabled, comment: 'PostgreSQL connection' }
    #  - { port: 9100/tcp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus node exporter connection' }
    #  - { port: 9100/udp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus node exporter connection' }
    #  - { port: 9187/tcp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus postgres exporter connection' }
    #  - { port: 9187/udp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus postgres exporter connection' }

    # tuned.conf directory
    postgresql_host_tuned_directory: "{% if ansible_distribution_major_version|int==6%}/etc/tune-profiles{%elif ansible_distribution_major_version|int==7 %}/usr/lib/tuned{% else %}/etc/tune-profiles{% endif %}"

    # nr of hugepages for a host
    postgresql_host_nr_hugepages: 100

    #
    # Services
    #

    # disable/enable motd
    postgresql_host_configure_motd: true

    # disable/enable ntp service
    postgresql_host_configure_ntp: true

    #
    # Users & groups
    #

    # postgres profile file
    postgresql_host_postgres_user_profile_file: '.bash_profile'

    # postgres user UID and GID
    postgresql_host_postgres_uid: 673
    postgresql_host_postgres_gid: 503

    # postgres user security limits settings
    postgresql_host_postgres_seclimits:
      - { name: 'soft nproc', value: 16384 }
      - { name: 'hard nproc', value: 16384 }
      - { name: 'soft nofile', value: 4096 }
      - { name: 'hard nofile', value: 65536 }
      - { name: 'soft stack', value: 10240 }
      - { name: 'hard stack', value: 32768 }
      - { name: 'soft memlock', value: "{{ ((0.9 * ansible_memtotal_mb)*1024)|round|int }}" }
      - { name: 'hard memlock', value: "{{ ((0.9 * ansible_memtotal_mb)*1024)|round|int }}" }

    # pirmary DBAs group (OS)
    postgresql_host_primary_dba_group: pgdbadm

    # all DBAs groups (OS)
    postgresql_host_dba_groups:
      - { group: "{{ postgresql_host_primary_dba_group }}", gid: 9000 }

    # list of DBA users (OS accounts)
    postgresql_host_dba_users: []
    #   - { username: "user1", uid: 9626315, primgroup: '{{ postgresql_host_primary_dba_group }}', othergroups: "{{ postgresql_host_primary_dba_group }}", comment: 'User1, user1@foo.com', key: '' }    

    # configure sudoers entries
    postgresql_host_configure_sudoers: true

    # sudoers config files (per user/group)
    postgresql_host_sudoers_drop_in_files: []
    #  - filename: "{{ postgresql_user }}"
    #    entries:
    #      - '{{ postgresql_user }} ALL=(nagios) NOPASSWD: ALL'
    #      - '{{ postgresql_user }} ALL=(root) NOPASSWD: ALL'
    #  - filename: "{{ postgresql_host_primary_dba_group }}"
    #    entries:
    #      - '%{{ postgresql_host_primary_dba_group }} ALL=(ALL) NOPASSWD: /bin/su - postgres, /bin/su -c postgres'
    #      - '%{{ postgresql_host_primary_dba_group }} ALL=({{ postgresql_user }}) NOPASSWD: ALL'
    #      - '%{{ postgresql_host_primary_dba_group }} ALL=(root) NOPASSWD: /usr/bin/less /var/log/messages'
    #      - '%{{ postgresql_host_primary_dba_group }} ALL=(nagios) NOPASSWD: /home/nagios/NaCl/Command.pl'
    #      - '%{{ postgresql_host_primary_dba_group }} ALL=(root) NOPASSWD: ALL'

    # files to be copied to remote host
    postgresql_host_files_to_copy:
      - { file: menu, dest: '/home/{{ postgresql_user }}', owner: '{{ postgresql_user }}', group: '{{ postgresql_group }}', mode: '0740' }

    #
    # Local filesystems
    #

    # VG used to store postgres data files
    postgresql_host_datavg: datavg

    # datavg and binvg mountpoint prefixes
    postgresql_host_datavg_mountpoint_prefix: '{{ postgresql_datavg_mountpoint_prefix }}'  # /postgres/data
    postgresql_host_binvg_mountpoint_prefix: '{{ postgresql_binvg_mountpoint_prefix }}'    # /postgres/apps

    # let ansible playbook to create VGs, PVs, LVs and filesystems
    postgresql_host_configure_host_disks: true

    # filesystem layout related to PostgreSQL instances
    postgresql_host_fs_layout:
      - vgname: binvg
        state: present
        filesystem:
          - { mntp: '{{ postgresql_host_binvg_mountpoint_prefix }}/homes', lvname: lvpghomes, lvsize: 6G, fstype: ext4, mode: '0755' }
          - { mntp: '{{ postgresql_host_binvg_mountpoint_prefix }}/dump', lvname: lvpgdump, lvsize: 40G, fstype: ext4, mode: '0755' }
          - { mntp: '{{ postgresql_host_binvg_mountpoint_prefix }}/diag', lvname: lvpgdiag, lvsize: 3G, fstype: ext4, mode: '0755' }
        disk:
          - { device: /dev/sdc, pvname: /dev/sdc }
      - vgname: "{{ postgresql_host_datavg }}"
        state: present
        filesystem:
          - { mntp: '{{ postgresql_host_datavg_mountpoint_prefix }}/archive', lvname: lvpgarchive, lvsize: 30G, fstype: ext4, mode: '0755' }
        disk:
          - { device: /dev/sdd, pvname: /dev/sdd }
      - vgname: backupvg
        state: present
        filesystem:
          - { mntp: /postgres/backup, lvname: lvpgbackup, lvsize: 99G, fstype: ext4, mode: '0755' }
        disk:
          - { device: /dev/sde, pvname: /dev/sde }

    # list of VG disks
    postgresql_host_fs_layout_vgdisks: "{%- for disk in item.disk -%} {{disk.pvname}} {%- if not loop.last -%},{%- endif -%}{% endfor %}"

    #
    # NFS
    #

    # configure NFS
    postgresql_host_configure_nfs: true

    # list of NFS servers
    postgresql_host_nfs_servers: []

    #
    # PostgreSQL Database Service
    #

    # configure dbpostgres service
    postgresql_host_configure_dbpostgres_service: true

    #
    # Customization
    #
	
    # enable host customization
    postgresql_host_customization_enable: false

    # tasks files
    postgresql_host_customization_tasks_from: []
	

Dependencies
------------

This role uses `postgresql` role.

Example Playbook
----------------

    - name: Configure PostgreSQL database server
      hosts: pg-servers
      become: true
      roles:
        - { role: postgresql-host, tags: postgresql_host }

Inside `vars/main.yml` or `group_vars/..` or `host_vars/..`:

    #--------------------------------------------
    # overrides role 'postgresql-host' variables
    #--------------------------------------------

    # disable/enable firewall service
    postgresql_host_disable_firewall: false

    # custom firewalld config
    postgresql_host_enable_custom_firewalld_config: true
    postgresql_host_custom_firewalld_config:
      - { port: 5432/tcp, permanent: true, immediate: true, state: enabled, comment: 'PostgreSQL connection' }
      - { port: 5432/udp, permanent: true, immediate: true, state: enabled, comment: 'PostgreSQL connection' }
      - { port: 9100/tcp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus node exporter connection' }
      - { port: 9100/udp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus node exporter connection' }
      - { port: 9187/tcp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus postgres exporter connection' }
      - { port: 9187/udp, permanent: true, immediate: true, state: enabled, comment: 'Prometheus postgres exporter connection' }

    # pirmary DBAs group (OS)
    postgresql_host_primary_dba_group: pgdbadm

    # list of DBA users (OS accounts)
    postgresql_host_dba_users:
       - { username: "user1", uid: 9626315, primgroup: '{{ postgresql_host_primary_dba_group }}', othergroups: "{{ postgresql_host_primary_dba_group }}", comment: 'User1, user1@foo.com', key: '<secret>' }

	       # sudoers config files (per user/group)
    postgresql_host_sudoers_drop_in_files:
      - filename: "{{ postgresql_user }}"
        entries:
          - '{{ postgresql_user }} ALL=(nagios) NOPASSWD: ALL'
          - '{{ postgresql_user }} ALL=(root) NOPASSWD: ALL'
      - filename: "{{ postgresql_host_primary_dba_group }}"
        entries:
          - '%{{ postgresql_host_primary_dba_group }} ALL=(ALL) NOPASSWD: /bin/su - postgres, /bin/su -c postgres'
          - '%{{ postgresql_host_primary_dba_group }} ALL=({{ postgresql_user }}) NOPASSWD: ALL'
          - '%{{ postgresql_host_primary_dba_group }} ALL=(root) NOPASSWD: /usr/bin/less /var/log/messages'
          - '%{{ postgresql_host_primary_dba_group }} ALL=(nagios) NOPASSWD: /home/nagios/NaCl/Command.pl'
          - '%{{ postgresql_host_primary_dba_group }} ALL=(root) NOPASSWD: ALL'

    #
    # Local filesystems
    #

    # VG used to store postgres data files
    postgresql_host_datavg: datavg

    #
    # Local filesystems
    #

    # VG used to store postgres data files
    postgresql_host_datavg: datavg

    # filesystem layout related to PostgreSQL instances
    postgresql_host_fs_layout:
      - vgname: binvg
        state: present
        filesystem:
          - { mntp: /postgres/apps/homes, lvname: lvpghomes, lvsize: 6G, fstype: ext4, mode: '0755' }
          - { mntp: /postgres/apps/dump, lvname: lvpgdump, lvsize: 40G, fstype: ext4, mode: '0755' }
          - { mntp: /postgres/apps/diag, lvname: lvpgdiag, lvsize: 3G, fstype: ext4, mode: '0755' }
        disk:
          - { device: /dev/sdc, pvname: /dev/sdc }
      - vgname: "{{ postgresql_host_datavg }}"
        state: present
        filesystem:
          - { mntp: /postgres/data/archive, lvname: lvpgarchive, lvsize: 30G, fstype: ext4, mode: '0755' }
        disk:
          - { device: /dev/sdd, pvname: /dev/sdd }
      - vgname: backupvg
        state: present
        filesystem:
          - { mntp: /postgres/backup, lvname: lvpgbackup, lvsize: 99G, fstype: ext4, mode: '0755' }
        disk:
          - { device: /dev/sde, pvname: /dev/sde }

    # enable host customization
    postgresql_host_customization_enable: true

    # tasks files
    postgresql_host_customization_tasks_from:
      - foo.yml

    # ... etc ...


License
-------

GPLv3 - GNU General Public License v3.0

Author Information
------------------

This role was created in 2018 by [Krzysztof Lewandowski](mailto:Krzysztof.Lewandowski@fastmail.fm).


