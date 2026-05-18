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
deploy@lima:~$ whoami
deploy
deploy@lima:~$ id
uid=1000(deploy) gid=1000(deploy) groups=1000(deploy)
```

# upload
rsync -aP -e "$SSH" --rsync-path="sudo rsync" \
    lima/  deploy@127.0.0.1:/opt/wp-deploy/

#Verify
```
$ls -la

drwxr-xr-x  5  501 staff 4096 May 18 13:48 wp-deploy    501 
--------------******----------------------
 - rsync ฝั่ง VM รันเป็น root (--rsync-path="sudo rsync") → มีสิทธิ์ chown →
  มันเลย set owner ตามไฟล์ต้นทาง
  - ต้นทาง = Mac ของคุณ user bblbook = uid 501, group staff
  - VM ไม่มี user uid 501 → ls เลยโชว์เลขดิบ 501
--------------******----------------------
deploy@lima-wp-prod:/opt$ sudo chown -R deploy:deploy wp-deploy

deploy@lima-wp-prod:/opt$ ls -la

drwxr-xr-x  5 deploy deploy 4096 May 18 13:48 wp-deploy
```