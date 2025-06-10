# Using SFTP to Connect to EC2 Server on Linux

## A)	by using SSH (Private Key .pem)
```bash
Make Sure the VPN is Connected
Select the appropriate VPN connection “CeqVPN-Amazon”
```

### Set Proper Permissions on the .pem Key File   ( 3shan login in vm in aws )
Run this commands : 
```bash
systemctl start sshd
icacls "C:\Users\HDsupport\Ec2-For-General-Purposes" /inheritance:r
icacls "C:\Users\HDsupport\Ec2-For-General-Purposes" /remove "Users" "INTERACTIVE" "Authenticated Users" "Everyone"
icacls "C:\Users\HDsupport\Ec2-For-General-Purposes" /grant:r "$($env:USERNAME
ssh -i  "path file.pem"   username@ip 
```

![image](https://github.com/user-attachments/assets/c8fcf7af-a37a-4761-81c7-28dd159cb836)
![image](https://github.com/user-attachments/assets/43562487-10e2-466e-a4be-c5210fcbdd4a)

###	Connect via SFTP   
```bash
sftp  -i "path file.pem"   username@ip
```
![image](https://github.com/user-attachments/assets/ad300f4a-d5c1-41c4-b4ea-facf9e17883f)


## B)	Username + Password

1-	Create user  

```bash
adduser    username
echo "passwd" | sudo passwd
```
	Username : ceq-sftp
	Password: Welcome@3030     

2-	Prepare the Home Directory for Chroot   ( owned by root )
```bash
sudo chown  root:root  /home/ ceq-sftp
sudo chmod 755  /home/ceq-sftp
```
3-	Create Upload Directory for File Operations  ( owned by  ceq-sftp)
```bash
sudo mkdir    /home/ ceq-sftp/upload
sudo chown   ceq-sftp:ceq-sftp   /home/ ceq-sftp/upload
sudo chmod  755   /home/ceq-sftp/upload
```

![image](https://github.com/user-attachments/assets/f3a8c1c2-3078-48fc-84cb-4bcd159087fb)

###	Edit SSH Configuration
```bash
PasswordAuthentication yes 
ChallengeResponseAuthentication no 
UsePAM yes

Match new User-name
  ChrootDirectory /home/New-username
  ForceCommand internal-sftp 
  AllowTcpForwarding no 
  X11Forwarding no 
```
![image](https://github.com/user-attachments/assets/de028005-ee9d-4d18-8984-2fc32780130e)


###	Restart SSH Service
###	Local host : 
```bash
systemctl restart sshd
sftp   username@ip    ( new username el ana 3mltlo add )
```
![image](https://github.com/user-attachments/assets/aabe3b1f-e5a7-44c5-9ec5-a3316f35aaa0)
