# tear down and rebuild

https://gist.github.com/marshyon/c0e65d4980fb8a7e6843a9a7cef89939

# list ip addresses and record 100 and 101

https://gist.github.com/marshyon/f67c2a7c424bae33a1d1fd9324369c1c


# update hosts to reflect new ip addresses

```
[k8s]
192.168.88.66
192.168.88.67

...

[k8smaster]
192.168.88.66

[k8smaster:vars]
hostname=cks-master

[k8sworker01]
192.168.88.67
```


# run ansible

```
source .env
ssh-add ~/.ssh/build_key
ssh-copy-id -i ~/.ssh/build_key 192.168.88.<ip1>
ssh-copy-id -i ~/.ssh/build_key 192.168.88.<ip2>
ansible -i hosts k8s -m ping
```
where `.env` has in it 

```
ansible_sudo=<sudo password>
```

run playbook
```
ansible-playbook -e "ansible_sudo_pass=${ansible_sudo}" site.yml
```
