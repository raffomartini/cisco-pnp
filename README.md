# cisco-pnp

## pnp server discovery
There are 3 main way for the device to discover the ztp server:

1. DHCP Option 43: I hate with passion because the documentation isn't super clear and I'm using it for AP discovery, therefore I didn't tested it 
2. DNS: The device will look for **pnpserver**.*localdomain*, I like this one better and it is what I ended up using
3. Cloud redirect: couldn't test it because my lab sits behind a proxy

### DNS discovery configuration
This is the DHCP configuration needed, note the DNS server and the **local domain**.
```
ip dhcp pool Vlan1
 network 10.102.1.0 255.255.255.0
 default-router 10.102.1.1
 dns-server 10.102.2.20
 domain-name secmob.local
```
(and remember to create an entry on the DNS server!!)

## Gotchas
#### 1- The out of band management interface in cat3650 and cat3850 doensn't work properly:
  * Before everest - 16.5.1a - no discovery at all is done using that interface
  * in 16.5.1a the interface will be down after the pnp operation  
  
   ```
   interface GigabitEthernet0/0
    vrf forwarding Mgmt-vrf
    ip address 10.102.1.48 255.255.255.0
    negotiation auto
    shutdown
   ```
   I have tried to force the `no shutdown` on the configuration, but this in turn made the pnp process fail AND shutted down the port anyway - **_investigations in process_**
   
#### 2- In 16.5.1a weird behaviours are observed if the configuration for the port used for pnp is not in switchport
  In my case this is the intended configuration:
```
interface GigabitEthernet1/0/37
 no switchport
 ip address 10.102.1.11 255.255.255.0
```
  And this is the outcome:
```
interface GigabitEthernet1/0/37
 no switchport
 no ip address
```
Workaround: use switcport and move the ip to the vlan.
