
出现每次连接虚拟机都需要输入私钥的口令：  

```
➜  Ansible ansible -i hosts localvm -a 'date'
Enter passphrase for key '/Users/ml/.ssh/local_vm_rsa': 
192.168.16.35 | SUCCESS | rc=0 >>
Fri Jun 16 06:00:48 UTC 2017
```

解决办法：  

```
ssh-add /Users/ml/.ssh/local_vm_rsa
Enter passphrase for /Users/ml/.ssh/local_vm_rsa: 
Identity added: /Users/ml/.ssh/local_vm_rsa (/Users/ml/.ssh/local_vm_rsa)
```

验证下：  

```
ansible -i hosts localvm -a 'date' 
192.168.16.35 | SUCCESS | rc=0 >>
Fri Jun 16 06:05:37 UTC 2017
```



