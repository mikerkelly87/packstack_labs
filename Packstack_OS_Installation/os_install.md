### These are the steps to install a minimal CentOS 8 OS for Packstack

**Select your language and click Continue**
![](screenshots/1.png)

**Click on Software Selection**
![](screenshots/2.png)

**Change the Base Environment to Minimal Install and click Done**
![](screenshots/3.png)

**Click on Installation Desintation**
![](screenshots/4.png)

**Select Custom for the Storage Configuration**
![](screenshots/5.png)

**Select the Standard Partition scheme**
![](screenshots/6.png)

**Click the + button and add the following partitions:**
```
1GB /boot
4GB swap
Remaining Space /

Click Done when finished
```
![](screenshots/7.png)

**Click on Network & Host Name**
![](screenshots/8.png)

**Switch your network adapter to ON and set the Host Name to packstack**
```
Also click on Configure and setup a static IP

Click Done when finished
```
![](screenshots/9.png)

**Click on Begin Installation**
![](screenshots/10.png)

**When the installation begins set a root password**
![](screenshots/11.png)



[<-- Back to Main](../README.md)