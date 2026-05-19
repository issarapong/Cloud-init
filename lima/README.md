## prepare disk
```
imactl disk create wp-data --size 20GiB
INFO[0000] Creating qcow2 disk "wp-data" with size 20GiB 
```
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

## mountdata disk
```
deploy@lima-wp-prod:~$  lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
NAME      SIZE TYPE FSTYPE  MOUNTPOINT
sr0     318.3M rom  iso9660 /mnt/lima-cidata
vda        20G disk         
├─vda1   19.9G part ext4    /
├─vda14     3M part         
└─vda15   124M part vfat    /boot/efi
vdb        20G disk  
```
* - vdb = data disk 20G, ช่อง FSTYPE ว่าง = ยังไม่มี filesystem → ตัวนี้แหละที่จะฟอร์แมต
## verify mounted
```
deploy@lima-wp-prod:~$ findmnt /opt 
[ถ้าผลลัพว่างคือยัง ไม่ mount folder /opt เข้ากับ vdb disk]
``` 
## ฟอร์แมต data disk เป็น ext4
```
  sudo mkfs.ext4 -L wp-data /dev/vdb
  - -L wp-data = ตั้ง label ชื่อ wp-data
  - ⚠️  คำสั่งนี้ล้างข้อมูลในดิสก์ทั้งลูก — รันเฉพาะตอนยืนยันแล้วว่า vdb ว่างจริง (จากขั้น 1)

deploy@lima-wp-prod:~$ sudo mkfs.ext4 -L wp-data /dev/vdb
mke2fs 1.47.2 (1-Jan-2025)
Discarding device blocks: done                            
Creating filesystem with 5242880 4k blocks and 1310720 inodes
Filesystem UUID: 919485bf-2f21-4f98-a71c-5fd23d9abdb5
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done   
```
## Veify format
```
deploy@lima-wp-prod:~$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
NAME      SIZE TYPE FSTYPE  MOUNTPOINT
sr0     318.3M rom  iso9660 /mnt/lima-cidata
vda        20G disk         
├─vda1   19.9G part ext4    /
├─vda14     3M part         
└─vda15   124M part vfat    /boot/efi
vdb        20G disk ext4    
```
##  prepaire mount 
  - จะได้ค่าเช่น a1b2c3d4-.... — จดไว้ ใช้ในขั้น 6
  - ใช้ UUID แทน /dev/vdb เพราะชื่อ device อาจสลับตอนบูต แต่ UUID ติดอยู่กับ filesystem เสมอ
```
sudo blkid -s UUID -o value /dev/vdb
919485bf-2f21-4f98-a71c-5fd23d9abdb5
# เพิ่มเข้า /etc/fstab
```
  UUID=919485bf-2f21-4f98-a71c-5fd23d9abdb5  /opt  ext4  defaults,noatime  0  2    

save
sudo systemctl daemon-reload
```
## เช็ค fstab ก่อน mount

  fstab เขียนผิด = บูตครั้งหน้าไม่ขึ้น (ตกไป emergency mode) — เช็คให้ผ่านก่อน:
  sudo findmnt --verify --verbose

ควรจบด้วย 0 errors, 0 warnings (หรือไม่มี error ที่บรรทัด /opt) — ถ้าผ่าน ค่อย mount:
```
## mount
```
$ sudo mount /opt
$ df -h /opt
Filesystem      Size  Used Avail Use% Mounted on
/dev/vdb         20G  2.1M   19G   1% /opt

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




# Q
❯ มี สั่ง mount แล้วไป เพิ่ม fstab auto ไหม   สั่ง mount /opt เฉยๆ ได้เพราะ /opt อยู่ใน fstab แล้ว —  
  mount ไปอ่าน device + options จาก fstab                                                  
    เอง                                                                                   

⏺ ไม่ครับ — mount ไม่เคยเขียน fstab เลย มันอ่านอย่างเดียว ทิศทางเดียว:

  fstab  ──อ่าน──>  mount        ✅ (mount ไปอ่าน fstab)
  fstab  <──เขียน── mount        ❌ ไม่มี — mount ไม่แตะ fstab

  ที่ fstab มีบรรทัด /opt = เพราะคุณพิมพ์เข้าไปเองด้วยมือตอนขั้น 6 (tee -a /etc/fstab) ไม่ใช่ mount
  ไปเพิ่มให้

  มันคือ 2 เรื่องแยกกัน — คนละหน้าที่

  ┌───────────────────────────┬───────────────────────────┬──────────────────────────┐
  │                           │          ทำอะไร           │       อยู่ได้นานแค่ไหน       │
  ├───────────────────────────┼───────────────────────────┼──────────────────────────┤
  │ sudo mount /opt (ขั้น 7)    │ mount เดี๋ยวนี้               │ แค่รอบบูตนี้ — reboot แล้วหลุด │
  ├───────────────────────────┼───────────────────────────┼──────────────────────────┤
  │ บรรทัดใน /etc/fstab (ขั้น 6) │ สั่งให้ mount อัตโนมัติทุกครั้งที่บูต │ ถาวร จนกว่าจะลบบรรทัด      │
  └───────────────────────────┴───────────────────────────┴──────────────────────────┘