# Linux 筆記

## EC2 set-up
> 操作web console的部分不屬於linux就不記錄在這

### Create new user
預設的username是ec2-user
所以第一次登入的指令應該是 ```ssh -i ec2.pem ec2-user@0.0.0.0```
如果不想要ec2-user作為username
可以創一個新的user，並把原本的ec2-user刪掉，也可以改變ec2-user的username
這次我採用前者
（以下指令環境為amazon linux 2，和centOS幾乎一樣）
```bash
$ sudo adduser james
$ sudo gpasswd -a james wheel
# now set up ssh
$ sudo mkdir /home/james/.ssh
$ sudo cp ~/.ssh/authorized_keys /home/james/.ssh/
$ sudo chown james /home/james/.ssh/authorized_keys
$ sudo chgrp james /home/james/.ssh/authorized_keys
# log in with just-created user
$ sudo userdel ec2-user
```
### Install docker

https://zhuanlan.zhihu.com/p/75959714

