<!SLIDE>
#Ansible Playbook
## Rolling App Server Restart

    @@@ puppet
    ---
    - hosts: "{{target}}:&~(.*app.*|.*wapp.*)"
      remote_user: psadm2
      serial: 1
      tasks:
        - name:    "Restart {{target}} application server and \
                   preloading cache using pscontrol"
          command: pscontrol --restart --force --configure --cleanipc \
                   --preload --wait --domain {{target}} --type app
          async:   300
          poll:    5
        - name: "Verifying that {{target}} Application Server is running"
          wait_for: "host={{ansible_nodename}} port={{jsl_port}}"

