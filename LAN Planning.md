# LAN Planning
## Address Inventory

**RFC 1918 Address Block: ** 192.168.0.0 – 192.168.255.255 

**Static: ** 20

**Leased: ** 108

**# Address Total: ** 128 (however, .0 can never be used, .1 is gateway, by convention, and .127 is broadcast)

## Additional Address Constraints

**CIDR Length: ** 25

**Subnet Mask: ** 255.255.255.128

**Network/Subnet ID: ** 192.168.174.0/25

**Broadcast Address: ** 192.168.174.127

## Address Assignments

**Default Gateway: ** 192.168.174.1

**DHCP Server: ** 192.168.174.1

**DHCP Lease Pool (IP Range - Two Values) : ** 192.168.174.23 - 192.168.174.73 (50 addresses)

**DNS Resolver: ** 75.75.75.75