# ansible FAQ

## FAQ

### ansible remote host No response

- [SSH works, but ansible throws unreachable error ](https://github.com/ansible/ansible/issues/15321)

- [paramiko](https://legacy.gitbook.com/book/yangdejie/paramiko/details)

  ```yml
  use paramiko as a workaround.

  ansible-playbook abc.yml -i development -c paramiko
  or add to ansible config

  [defaults]
  transport = paramiko
  ```

- 检查互信状态
  - Limiting ssh key permissions to 600 fixed this issue.
  - I was using a non-standard private key and it wasn't being found. It was 600 perms. ssh-add <path-to-private-key> fixed my issue.
  - Adding my key with ssh-copy-id to the remote server fixed the problem.


```yml
for me, this resolved on Ubuntu 16.04 with this added to ansible.cfg:

[ssh_connection] 
# for running on Ubuntu
control_path=%(directory)s/%%h-%%r
On related note, this resolved it when running on Mac host:

[ssh_connection] 
# for running on OSX
control_path = %(directory)s/%%C
```

```bash
Solved the issue by installing sshpass using command:

sudo apt-get install sshpass
```