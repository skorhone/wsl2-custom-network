# wsl2-custom-network
This project contains powershell scripts for creating custom network for WSL2

This project contains code fragments from https://www.powershellgallery.com/packages/HNS/0.2.4

# Prologue
I like many others wasted half a day trying to figure out how to get WSL2 to run properly in Enterprise environment. I was badly mistaken when I thought that there's a configuration option for WSL2's network.

WSL2 chooses subnet from [private network ranges](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses) using "clever" logic that prevents collisions with known private networks. It works great on your home workstation where you're connected directly to your private network. Unfortunately this logic breaks in almost every enterprise network. In most enterprises workstations have no visibility to reserved private network segments - which means that WSL2 has no visibility when it creates subnet configuration. Considering subnet that WSL2 creates is both random and huge (/20), it's extremly likely to eventually collide with existing subnets and prevent user from accessing services. I was actually not able to get it generate valid configuration after numerous reboots.

The second issue is that WSL2 resets network configuration on every reboot. The end result is that network configuration is more or less random. To use network from WSL2 one must allow traffic from and to all network ranges that WSL2 *might* use. This might be acceptable for some users, but most enterprises enforce strict network security policies.

And as you might expect (with Microsoft products), I could find no meaningful documentation on how WSL2 manages the network configuration. I spent half a day reading posts on GitHub only to find that no-one seems to know. There was a lot of discussions on virtual switches and managing network interfaces. After some debugging it became evident that none of the proposed solutions/hacks work reliably enough.

I was just about to quit and revert back to VirtualBox when I came across a bit of information that pointed to the right direction. Few google searches later I was convinced that WSL2 uses Host Compute Network (HCN) Microsoft advertised API as public and documentation looked promising. Unfortunately documentation is outdated and missing a lot of fine details. Fortunately HCN api handles requests and responses in JSON format, so I was able to reverse engineer data structures that were require to create WSL2 compatible network configuration.

So here we have it. A script that creates a WSL2 compatible network using Host Computer Network API.

# Usage
Please notice that you most certainly want to edit the sample network configuration. Would you like to use NAT over ICS (google for difference), set Type to NAT and Flags to 0.

I've found that DNSServerList doesn't seem to have any effect. In NAT mode there is no local DNS server, so you need to edit your /etc/resolv.conf and add DNS server that works for your environment.

1. Install WSL2
2. Download contents of this repository
3. Open PowerShell as administractor change to git repository location and execute:
```
import-module -Name .\hcn

$network = @"
{
        "Name" : "WSL",
        "Flags": 9,
        "Type": "ICS",
        "IPv6": false,
        "IsolateSwitch": true,
        "MaxConcurrentEndpoints": 1,
        "Subnets" : [
            {
                "ID" : "FC437E99-2063-4433-A1FA-F4D17BD55C92",
                "ObjectType": 5,
                "AddressPrefix" : "192.168.143.0/24",
                "GatewayAddress" : "192.168.143.1",
                "IpSubnets" : [
                    {
                        "ID" : "4D120505-4143-4CB2-8C53-DC0F70049696",
                        "Flags": 3,
                        "IpAddressPrefix": "192.168.143.0/24",
                        "ObjectType": 6
                    }
                ]
            }
        ],
        "MacPools":  [
            {
                "EndMacAddress":  "00-15-5D-52-CF-FF",
                "StartMacAddress":  "00-15-5D-52-C0-00"
            }
        ],
        "DNSServerList" : "192.168.5.1, 192.168.5.2"
}
"@

Get-HnsNetworkEx | Where-Object { $_.Name -Eq "WSL" } | Remove-HnsNetworkEx
New-HnsNetworkEx -Id B95D0C5E-57D4-412B-B571-18A81A16E005 -JsonString $network
```
4. Once you're happy with the configuration, set up a startup script that creates WSL2 network during startup.