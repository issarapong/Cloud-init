## start lima
```
limactl start --name=wp-prod lima.yaml
```
## ssh
```
ssh -i ~/.ssh/lima -p 2222 deploy@127.0.0.1
```
## verify
```
deploy@lima-wp-prod:~$ whoami
deploy
deploy@lima-wp-prod:~$ id
uid=1000(deploy) gid=1000(deploy) groups=1000(deploy)
```