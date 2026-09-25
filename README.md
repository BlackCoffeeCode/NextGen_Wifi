# D-Link DAP-X3060 + FreeRADIUS + OpenLDAP Lab Setup

This repository documents the working lab configuration for a D-Link DAP-X3060 wireless access point authenticating users through FreeRADIUS backed by OpenLDAP.

The setup was validated and reproduced as a single-laptop lab environment using a Kali VM and a local relay on the main Mac.

---

## 1. Final architecture

```text
                         CAMPUS NETWORK
                         172.17.28.0/24
                               │
                ┌──────────────┴──────────────┐
                │                             │
        D-Link DAP-X3060                  Main Mac
        172.17.28.237                     en7
                │                         172.17.28.124
                │                             │
                │                         socat :1812
                │                             │
                │                             ▼
                │                         Kali VM
                │                      172.16.235.129
                │                             │
                │                        FreeRADIUS
                │                             │
                │                         OpenLDAP
                │                         127.0.0.1:389
                │
                ▼
        CollegeWifi-Test-5
                │
                ▼
          Test Device
```

The DAP-X3060's LAN/PoE port is the normal Ethernet/network connection; its separate RJ45 console port is for debug. The manual also documents connecting the AP to a PoE switch through the LAN(PoE) port.

### Addresses used in the lab

| Component | Address |
| --- | --- |
| D-Link DAP-X3060 | 172.17.28.237 |
| Main Mac, campus Ethernet en7 | 172.17.28.124 |
| Kali NAT interface | 172.16.235.129 |
| Mac VMware-side interface | 172.16.235.1 |
| OpenLDAP | 127.0.0.1:389 |
| FreeRADIUS | 0.0.0.0:1812 |
| LDAP Base DN | dc=campus,dc=test |
| LDAP users OU | ou=people |
| Test user | testuser |

---

## 2. OpenLDAP installation

### Install packages

On Kali:

```bash
sudo apt update
sudo apt install slapd ldap-utils
```

During installation, if needrestart asks:

```text
Restart services during package upgrades without asking?
```

Select:

```text
Yes
```

### Configure LDAP

Run:

```bash
sudo dpkg-reconfigure -plow slapd
```

Use:

```text
Omit OpenLDAP server configuration?   No
DNS domain name:                      campus.test
Organization name:                   Campus Lab
Database backend:                    MDB
Remove database when slapd is purged: No
```

Set a known LDAP administrator password.

This gives:

```text
Base DN:
dc=campus,dc=test

Administrator:
cn=admin,dc=campus,dc=test
```

### Start LDAP

```bash
sudo systemctl enable --now slapd
```

Check:

```bash
sudo systemctl status slapd
```

It should say:

```text
Active: active (running)
```

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%202.56.09%20PM.png" alt="OpenLDAP install screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.01.37%20PM.png" alt="OpenLDAP configuration screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.03.00%20PM.png" alt="OpenLDAP service status screenshot" width="1000" />
</div>

---

## 3. Verify OpenLDAP

Check the directory suffix:

```bash
ldapsearch -x -LLL -s base -b "dc=campus,dc=test" namingContexts
```

Expected:

```text
dn: dc=campus,dc=test
```

Test the LDAP administrator:

```bash
ldapwhoami -x -H ldap://127.0.0.1 \
-D "cn=admin,dc=campus,dc=test" -W
```

Expected:

```text
dn:cn=admin,dc=campus,dc=test
```

---

## 4. Create the LDAP users OU

Create:

```bash
nano ~/ldap-base.ldif
```

Put:

```ldif
dn: ou=people,dc=campus,dc=test
objectClass: organizationalUnit
ou: people
```

Add it:

```bash
ldapadd -x -H ldap://127.0.0.1 \
-D "cn=admin,dc=campus,dc=test" \
-W -f ~/ldap-base.ldif
```

---

## 5. Create the test user

Generate an SSHA password:

```bash
slappasswd
```

Enter:

```text
TestPassword123!
```

Copy the resulting `{SSHA}...` value.

Create:

```bash
nano ~/testuser.ldif
```

Use:

```ldif
dn: uid=testuser,ou=people,dc=campus,dc=test
objectClass: inetOrgPerson
cn: Test User
sn: User
uid: testuser
userPassword: {SSHA}PASTE_HASH_HERE
```

Replace `{SSHA}PASTE_HASH_HERE` with the actual `slappasswd` output.

Add it:

```bash
ldapadd -x -H ldap://127.0.0.1 \
-D "cn=admin,dc=campus,dc=test" \
-W -f ~/testuser.ldif
```

---

## 6. Test the LDAP user directly

Run:

```bash
ldapwhoami -x \
-H ldap://127.0.0.1 \
-D "uid=testuser,ou=people,dc=campus,dc=test" \
-W
```

Enter:

```text
TestPassword123!
```

Expected:

```text
dn:uid=testuser,ou=people,dc=campus,dc=test
```

At this point:

```text
OpenLDAP
   │
   └── testuser ✅
```

Do not proceed until this works.

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%203.05.04%20PM.png" alt="LDAP user verification screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.19.10%20PM.png" alt="LDAP user search screenshot" width="1000" />
</div>

---

## 7. Install FreeRADIUS

On Kali:

```bash
sudo apt update
sudo apt install freeradius freeradius-ldap freeradius-utils
```

Check:

```bash
freeradius -v
```

---

## 8. Enable the LDAP module

Normally:

```bash
ls -l /etc/freeradius/3.0/mods-enabled/ldap
```

It should point to the module in `mods-available`.

If needed:

```bash
sudo ln -s ../mods-available/ldap \
/etc/freeradius/3.0/mods-enabled/ldap
```

Be careful not to create a self-referencing symlink. The correct relationship is:

```text
mods-enabled/ldap
        ↓
mods-available/ldap
```

---

## 9. Configure FreeRADIUS LDAP

Edit:

```bash
sudo nano /etc/freeradius/3.0/mods-enabled/ldap
```

The important settings for this lab are:

```text
ldap {
    server = "ldap://127.0.0.1"
    port = 389

    identity = "cn=admin,dc=campus,dc=test"
    password = "YOUR_LDAP_ADMIN_PASSWORD"

    base_dn = "dc=campus,dc=test"

    user {
        base_dn = "ou=people,dc=campus,dc=test"
        filter = "(uid=%{%{Stripped-User-Name}:-%{User-Name}})"
    }
}
```

### Important

Use:

```text
ou=people,dc=campus,dc=test
```

as the user's search base.

We initially used:

```text
ou=people,${..base_dn}
```

and FreeRADIUS interpreted that literally in our setup, producing:

```text
Invalid DN syntax
```

Using the complete DN fixed the LDAP lookup.

---

## 10. Add LDAP to the authorization sections

Edit:

```bash
sudo nano /etc/freeradius/3.0/sites-enabled/default
```

Inside:

```text
authorize {
```

ensure:

```text
ldap
```

appears before `pap`.

Do the same in:

```bash
sudo nano /etc/freeradius/3.0/sites-enabled/inner-tunnel
```

So the relevant flow is:

```text
authorize {
    ...
    ldap
    ...
    pap
    ...
}
```

This is particularly important for EAP-TTLS/PAP.

FreeRADIUS documents LDAP as a database backend and recommends retrieving the known-good password from LDAP and letting PAP perform the authentication. FreeRADIUS also specifically recommends PAP as the inner method when initially testing EAP-TTLS.

---

## 11. Configure the RADIUS client

Edit:

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

Keep the existing localhost client:

```text
client localhost {
    ipaddr = 127.0.0.1
    secret = testing123
}
```

Add the Mac relay:

```text
client mac_radius_relay {
    ipaddr = 172.16.235.1
    secret = testing123
    shortname = mac-relay
}
```

### Why 172.16.235.1?

Because the Mac's socat relay connects into the Kali VM from its VMware-side address. Your successful socat connection showed the local source as:

```text
172.16.235.1
```

FreeRADIUS requires the source address of a RADIUS client and its shared secret to be defined in `clients.conf`.

---

## 12. Validate FreeRADIUS

Run:

```bash
sudo freeradius -XC
```

You want:

```text
Configuration appears to be OK
```

---

## 13. Start FreeRADIUS

For normal operation:

```bash
sudo systemctl start freeradius
```

Check:

```bash
sudo systemctl status freeradius --no-pager
```

Check UDP/1812:

```bash
sudo ss -lunp | grep 1812
```

Expected:

```text
0.0.0.0:1812
[::]:1812
```

---

## 14. Test FreeRADIUS against LDAP

Before involving the D-Link:

```bash
radtest testuser 'TestPassword123!' 127.0.0.1 0 testing123
```

Expected:

```text
Received Access-Accept
```

This proves:

```text
radtest
   ↓
FreeRADIUS
   ↓
OpenLDAP
```

Your successful test ultimately showed:

```text
LDAP user found
        ↓
User authenticated successfully
        ↓
Access-Accept
```

That is your first complete authentication checkpoint.

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%203.26.03%20PM.png" alt="FreeRADIUS validation screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.27.16%20PM.png" alt="FreeRADIUS config test screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.28.18%20PM.png" alt="FreeRADIUS local auth screenshot" width="1000" />
</div>

---

## 15. Main Mac network configuration

Your final main Mac network is:

```text
en7 = 172.17.28.124
```

Verify:

```bash
ipconfig getifaddr en7
```

Expected:

```text
172.17.28.124
```

Verify the AP is reachable:

```bash
ping -c 4 172.17.28.237
```

Expected:

```text
0% packet loss
```

Your Mac's Ethernet interface `en7` is therefore the interface exposed to the D-Link/campus network.

---

## 16. Install socat on the main Mac

If not installed:

```bash
brew install socat
```

Verify:

```bash
socat -V
```

Check whether UDP/1812 is free:

```bash
sudo lsof -nP -iUDP:1812
```

No output is good.

---

## 17. Start the RADIUS relay on the Mac

Use this working command:

```bash
sudo socat -d -d \
UDP4-LISTEN:1812,bind=172.17.28.124,reuseaddr \
UDP4:172.16.235.129:1812
```

Do not add `fork` in this particular lab setup. We initially used it and hit:

```text
Address already in use
```

The no-fork version successfully received the D-Link RADIUS connection.

Successful output looked like:

```text
listening on UDP ... 172.17.28.124:1812
accepting UDP connection from ... 172.17.28.237:xxxxx
opening connection to 172.16.235.129:1812
successfully connected to 172.16.235.129:1812
starting data transfer loop
```

That proves:

```text
D-Link → Mac → Kali
```

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%203.32.41%20PM.png" alt="socat relay screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.37.50%20PM.png" alt="Mac TCP listener screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.40.57%20PM.png" alt="socat and port verification screenshot" width="1000" />
</div>

---

## 18. D-Link RADIUS configuration

Open:

```text
https://172.17.28.237
```

The DAP-X3060's web interface supports WPA-Enterprise and external RADIUS. The manual specifies the RADIUS server address, port and shared secret, and says to use Save → Configuration → Save and Activate to make the configuration permanent.

For the test SSID:

```text
CollegeWifi-Test-5
```

use:

| Setting | Value |
| --- | --- |
| Wireless Band | 5 GHz |
| Mode | Access Point |
| SSID | CollegeWifi-Test-5 |
| Authentication | WPA-Enterprise |
| WPA Mode | WPA2 Only |
| Cipher Type | AES |
| Group Key Update | 3600 |
| Network Access Protection | Disable |
| RADIUS Server Mode | External |
| RADIUS Server | 172.17.28.124 |
| RADIUS Port | 1812 |
| RADIUS Secret | testing123 |
| Backup RADIUS | Empty |
| Accounting Mode | Disable |

Then:

```text
Save
→ Configuration
→ Save and Activate
```

Repeat the same security/RADIUS configuration for the 2.4 GHz SSID if you want both radios to use RADIUS.

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%203.46.12%20PM.png" alt="D-Link web UI screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%203.53.50%20PM.png" alt="D-Link security config screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%204.06.17%20PM.png" alt="D-Link radius config screenshot" width="1000" />
</div>

---

## 19. Very important: RADIUS IP on the D-Link

The D-Link uses:

```text
172.17.28.124
```

not:

```text
172.16.235.129
```

because:

```text
D-Link
172.17.28.237
     ↓
Mac
172.17.28.124:1812
     ↓
socat
     ↓
Kali
172.16.235.129:1812
```

The D-Link only needs to know where socat is listening.

---

## 20. Test Wi-Fi authentication

On Kali, stop the background service:

```bash
sudo systemctl stop freeradius
```

Start debug mode:

```bash
sudo freeradius -X
```

Wait for:

```text
Ready to process requests
```

On the main Mac, keep:

```bash
sudo socat -d -d \
UDP4-LISTEN:1812,bind=172.17.28.124,reuseaddr \
UDP4:172.16.235.129:1812
```

running.

Then connect the test device to:

```text
CollegeWifi-Test-5
```

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%206.00.43%20PM.png" alt="RADIUS debug startup screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.09.33%20PM.png" alt="RADIUS request debug screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.09.42%20PM.png" alt="Wi-Fi auth attempt screenshot" width="1000" />
</div>

---

## 21. Recommended client authentication method

For this LDAP implementation, use:

```text
EAP-TTLS
    ↓
PAP
    ↓
LDAP
```

rather than:

```text
PEAP
    ↓
MSCHAPv2
```

### Why?

Your LDAP user password is stored as:

```text
{SSHA}...
```

PAP works because FreeRADIUS can retrieve the LDAP password and compare it. FreeRADIUS's documentation states that normal LDAP password retrieval is compatible with PAP and recommends EAP-TTLS/PAP when passwords are stored in hashed LDAP formats.

We tested PEAP/MSCHAPv2 and the server reported:

```text
No NT-Password. Cannot perform authentication
```

So PEAP/MSCHAPv2 was not compatible with the way this test user's password was stored.

---

## 22. macOS test client

On the test Mac, configure:

```text
SSID:
CollegeWifi-Test-5

Security:
WPA2-Enterprise

EAP:
TTLS

Inner authentication:
PAP

Username:
testuser

Password:
TestPassword123!
```

macOS may require a configuration profile to expose/select TTLS/PAP cleanly. Apple supports EAP-TTLS in Wi-Fi configuration profiles, including PAP as the TTLS inner authentication method.

For future deployment, certificate validation should be configured properly rather than simply accepting an unknown certificate.

---

## 23. What you should see in FreeRADIUS

When the test client connects, the debug terminal should show:

```text
Received Access-Request
Then:
EAP-TTLS
Then inside the tunneled request:
PAP
Then LDAP:
Searching:
ou=people,dc=campus,dc=test
Then:
uid=testuser,ou=people,dc=campus,dc=test
Then:
User authenticated successfully
And finally:
Sent Access-Accept
```

FreeRADIUS documents that successful EAP-TTLS authentication should involve several Access-Request/Access-Challenge exchanges followed by an Access-Accept.

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%206.16.39%20PM.png" alt="Access-Request debug screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.19.44%20PM.png" alt="TTLS PAP debug screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.19.51%20PM.png" alt="Auth success screenshot" width="1000" />
</div>

---

## 24. Complete request flow

Your final working authentication chain is:

```text
              1. Wi-Fi association
Test Mac ----------------------------> D-Link
                                        |
                                        | 2. RADIUS Access-Request
                                        | UDP/1812
                                        v
                                  Main Mac
                                  172.17.28.124
                                        |
                                        | socat
                                        v
                                  Kali VM
                                  172.16.235.129
                                        |
                                        | 3. LDAP query
                                        v
                                  OpenLDAP
                                  127.0.0.1:389
                                        |
                                        | 4. User/password result
                                        v
                                  FreeRADIUS
                                        |
                                        | 5. Access-Accept
                                        v
                                  D-Link
                                        |
                                        v
                              Test Mac authenticated
```

---

## 25. Standard troubleshooting commands

### Check LDAP

```bash
sudo systemctl status slapd
sudo ss -lntp | grep 389
ldapsearch -x -LLL -s base -b "dc=campus,dc=test" namingContexts
```

### Check FreeRADIUS

```bash
sudo freeradius -XC
sudo systemctl status freeradius --no-pager
sudo ss -lunp | grep 1812
```

### Debug

```bash
sudo systemctl stop freeradius
sudo freeradius -X
```

### Check Mac → D-Link

```bash
ping -c 4 172.17.28.237
```

### Check Mac RADIUS listener

```bash
sudo lsof -nP -iUDP:1812
```

### Check incoming RADIUS packets

On the main Mac:

```bash
sudo tcpdump -ni en7 'udp port 1812'
```

You should see traffic from:

```text
172.17.28.237
```

### Check the FreeRADIUS relay traffic

On Kali:

```bash
sudo tcpdump -ni any 'udp port 1812'
```

---

## 26. Useful testing sequence

For future rebuilds, always test in this order:

```text
1. LDAP
   ↓
2. LDAP user authentication
   ↓
3. FreeRADIUS config validation
   ↓
4. FreeRADIUS → LDAP using radtest
   ↓
5. Mac → D-Link IP connectivity
   ↓
6. socat listener
   ↓
7. D-Link → Mac → Kali RADIUS packet
   ↓
8. EAP-TTLS/PAP Wi-Fi authentication
```

That way, when something breaks, you know exactly which layer is responsible.

---

## 27. Important lab-vs-production distinction

This setup is good for your single-laptop testing lab, but I would not deploy the socat relay architecture as the final college network design.

For production, the cleaner architecture is:

```text
D-Link
   │
   │ UDP 1812
   ▼
FreeRADIUS server
   │
   │ LDAP/LDAPS
   ▼
LDAP server
```

with FreeRADIUS directly reachable on the network.

Also, `testing123` is being used here as a lab shared secret. For an actual deployment, use a long random secret; FreeRADIUS's documentation explicitly treats the RADIUS shared secret as a security-critical value.

---

## One-page final configuration

For your exact lab, the values to keep in your notes are:

```text
================ LDAP ================

Domain:
campus.test

Base DN:
dc=campus,dc=test

Admin DN:
cn=admin,dc=campus,dc=test

Users OU:
ou=people,dc=campus,dc=test

Test User:
uid=testuser,ou=people,dc=campus,dc=test

Test Password:
TestPassword123!


============= FreeRADIUS =============

RADIUS:
UDP/1812

LDAP:
127.0.0.1:389

RADIUS relay client:
172.16.235.1

RADIUS shared secret:
testing123


================ Mac =================

Campus Ethernet:
en7

Mac IP:
172.17.28.124

socat:
172.17.28.124:1812
        ↓
172.16.235.129:1812


=============== D-Link ================

DAP-X3060:
172.17.28.237

RADIUS Server:
172.17.28.124

RADIUS Port:
1812

RADIUS Secret:
testing123

Authentication:
WPA-Enterprise

WPA Mode:
WPA2 Only

Cipher:
AES

Group Key:
3600

Accounting:
Disabled


============= Wi-Fi Client ============

SSID:
CollegeWifi-Test-5

EAP:
TTLS

Inner Auth:
PAP

Username:
testuser

Password:
TestPassword123!
```

That is the complete reproducible sequence for the lab you built.

---

## 5-Step startup routine

Since everything is already configured, your five-step startup procedure is:

1. Start LDAP — Kali

```bash
sudo systemctl start slapd
```

Verify:

```bash
sudo systemctl status slapd --no-pager
```

2. Start FreeRADIUS — Kali

For testing/debugging, use:

```bash
sudo systemctl stop freeradius
sudo freeradius -X
```

Wait for:

```text
Ready to process requests
```

Leave this terminal open.

3. Start the RADIUS listener/relay — Main Mac

Your Mac's current Ethernet IP is:

```text
172.17.28.124
```

Run:

```bash
sudo socat -d -d \
UDP4-LISTEN:1812,bind=172.17.28.124,reuseaddr \
UDP4:172.16.235.129:1812
```

Leave this terminal open.

You should see:

```text
listening on UDP ... 172.17.28.124:1812
```

4. Verify the RADIUS path

On the main Mac, in another terminal:

```bash
sudo lsof -nP -iUDP:1812
```

You should see `socat`.

On Kali, in another terminal:

```bash
sudo ss -lunp | grep 1812
```

You should see FreeRADIUS on:

```text
0.0.0.0:1812
```

Also test FreeRADIUS + LDAP locally:

```bash
radtest testuser 'TestPassword123!' 127.0.0.1 0 testing123
```

A successful result should include:

```text
Received Access-Accept
```

5. Connect the Wi-Fi client

Connect the test device to:

```text
CollegeWifi-Test-5
```

Use:

```text
Username: testuser
Password: TestPassword123!
EAP:      TTLS
Inner:    PAP
```

Watch the `freeradius -X` terminal.

The successful chain is:

```text
Test device
    ↓
D-Link 172.17.28.237
    ↓ UDP 1812
Mac 172.17.28.124
    ↓ socat
Kali 172.16.235.129
    ↓
FreeRADIUS
    ↓
LDAP
```

That is the complete 5-step startup routine, with the local `radtest` verification included.

### Screenshots

<div align="center">
  <img src="Screenshot%202026-09-25%20at%206.35.57%20PM.png" alt="final startup screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.37.23%20PM.png" alt="startup validation screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.39.03%20PM.png" alt="final Wi-Fi test screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.41.01%20PM.png" alt="successful authentication screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%206.44.46%20PM.png" alt="final success screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%207.25.08%20PM.png" alt="network validation screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%208.03.38%20PM.png" alt="lab environment screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%208.23.29%20PM.png" alt="terminal screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%208.23.45%20PM.png" alt="RADIUS relay screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%208.23.56%20PM.png" alt="Kali and Mac screenshot" width="1000" />
  <br /><br />
  <img src="Screenshot%202026-09-25%20at%209.44.37%20PM.png" alt="final validation screenshot" width="1000" />
  <br /><br />
  <img src="screenshots/WhatsApp%20Image%202026-09-26%20at%2000.14.02.jpeg" alt="WhatsApp final screenshot" width="1000" />
</div>

---

## Summary

This lab demonstrates a complete, working chain of:

```text
Wi-Fi client -> D-Link AP -> Mac relay -> Kali VM -> FreeRADIUS -> OpenLDAP -> Access-Accept
```

The key success factors were:

- Correct LDAP DN and OU configuration
- FreeRADIUS LDAP section ordering
- Matching RADIUS client IP and secret
- Using EAP-TTLS with PAP for hashed LDAP credentials
- Proper socat relay on the main Mac

This is the exact reproducible process used for the final working lab setup.
