#cloud-config
ssh_pwauth: true

chpasswd:
  expire: false
  users:
    - name: root
      password: "jhkimhomqwe!@#"
      type: text    