From the Ansible yum module documentation:

You can also pass a url or a local path to a rpm file. To operate on several packages this can accept a comma separated list of packages or (as of 2.0) a list of packages.

The proper steps to do this would be something like this:

- name: Copy rpm file to server
  copy:
     src: package.rpm
     dest: /tmp/package.rpm

- name: Install package.
  yum:
     name: /tmp/package.rpm
     state: present

Actually the yum module can install an RPM directly from a given URL:

- name: Remote RPM install with yum
  yum: name=http://example.com/some_package.rpm

- name: mkdir /tmp/RPMS
  file: path=/tmp/RPMS state=directory

- name: copy RPMs to /tmp/RPMS
  copy:
    src: "{{ item }}"
    dest: /tmp/RPMS
  with_fileglob:
    - "../files/*.rpm"
  register: rpms_copied

- name: local RPMs not found
  fail:
    msg: "RPMs not found in ../files/"
  when: rpms_copied.results|length == 0 and rpms_copied.skipped and rpms_copied.skipped_reason.find('No items') != -1

- set_fact:
    rpm_list: "{{ rpms_copied.results | map(attribute='dest') | list}}"

- name: install RPMs
  yum:
    name: "{{rpm_list}}"

--------------

like to share my solution for installing a specific rpm version and only upload the rpm if neccessary.

---
- hosts: testhosts
  tasks:
          - name: check if package installed
            command: rpm -q --qf "%{VERSION}\n" package
            args:
                    warn: false
            register: package_installed
            failed_when: false
            changed_when: package_installed.rc != 0
          - name: check package version
            shell: "[[ `rpm -q --qf \"%{VERSION}\n\" package` == '{{ package_version }}' ]]"
            args:
                    warn: false
            register: package_installed_version
            failed_when: false
            changed_when: package_installed_version.rc != 0
          - name: uninstall old package
            yum:
                    name: package
                    state: removed
            when: package_installed_version.rc != 0 and package_installed.rc == 0
          - name: copy package-rpm to /tmp
            copy:
                    src: package-{{ package_version }}-1.el7.x86_64.rpm
                    dest: /tmp/
            notify:
                    - remove package-rpm from /tmp
            when: package_installed_version.rc != 0
          - name: install package
            yum:
                    name: /tmp/package-{{ package_version }}-1.el7.x86_64.rpm
                    state: present
            when: package_installed_version.rc != 0
  handlers:
          - name: remove package-rpm from /tmp
            file:
                    name: /tmp/package-{{ package_version }}-1.el7.x86_64.rpm
                    state: absent

- command: rpm -U "{{rpm}}"
  args:
    warn: false
  register: rpm_install
  failed_when: rpm_install.rc != 0 and "is already installed" not in rpm_install.stderr
  changed_when: rpm_install.rc == 0

- name: Installed the rpm files
  shell: yum localinstall *.rpm -y
  args:
    chdir: /tmp/rpm_dir


