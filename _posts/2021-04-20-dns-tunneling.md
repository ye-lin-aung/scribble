---
title: DNS Tunneling
date: '2021-04-20 10:36:49'
---

Since the coup, the military regime have been trying so hard to censor the internet in Myanmar. They block social media websites like twitter and facebook to censor whats happening in Myanmar. In march, military completely shutdown mobile data network and all networks to shutdown during 1 AM to 9 AM.  And a lot of developers have found many ways to bypass the censorship. We can buy some servers in whitelisted IP of specific provider and host VPN on it and connect to VPN. We can use specific port for specific provider. Most are using outline VPN to bypass the censorship but it doesn't work in many situation. People will start hosting outline VPNs in whitelisted IPs and providers will try to block the ip. [Hosting Outline VPN](https://medium.com/privacytools/self-hosting-a-shadowsocks-vpn-with-outline-274a956e6e08) is easy and you can find a lot of tutorials online. What I am going to show is another method of using internet during censorship, DNS tunneling. You need to know  [how DNS works](https://www.cloudflare.com/learning/dns/what-is-dns/)  before procceding the reading.

## The beginning

I noticed about DNS tunneling while I was trying different method to connect internet in complete shutdown networks. I tried things like pinging, port scanning and querying DNS. Strangely although pinging and ports are filtered/blocked, DNS queries are passing successfully. You can see DNS is returning the correct record while scanning. I sometimes use my android tablet to scan things while I am away from laptop. The tool I used below is [Net Anayzer](https://techet.net/netanalyzer).

![DNS Tunneling](/images/dns.jpg)

Ok. My first idea in mind is that if DNS server is able to query and able to response. What if we encode the whole tcp request as DNS query like <encoded_request>.a.com and send to name server and name server will response <encoded_response>.a.com as a record. I believe the idea is possible and someone have already done it so I do my research and found that its called DNS Tunneling. People used this to bypass firewalls and limitations in public networks. Although it works, its a very slow since there are a lot of encoding, decoding, request response cycles to query a single request. Attackers also use this for various attacks. 

## Tooling 
There are a lot of tools for DNS tunneling. I decided to use [iodine](https://code.kryo.se/iodine/) since its the most easiest way to setup on both server and client.

## Setting up server
You need to have following things for server configurations.
1. Any domain name. Its better its a domain that haven't used before since we are going to point name server to another location. 
2. A public accessible Linux server with SSH port and DNS port enabled. I used [AWS lightsail](https://aws.amazon.com/lightsail/) with Ubuntu 20.0.4.

Let's start the exciting part. 

1. Login to your DNS provider 
2. Point A record with domain name ds.mydomain.com  → public IP address
3. Point NS record to that domain n.mydomain.com  → ds.mydomain.com. 
Its better to use short domain for NS record since it allows as to add more data in DNS queries. 
4. Take a break. DNS take times to propogate. 
5. Test the DNS records with [Dig](https://toolbox.googleapps.com/apps/dig/). 

DNS can take up to 24 hours to propogate depending on your TTL configs and depending on your DNS provider. It something don't works, blame the DNS. 

![It is DNS](/images/always_dns.png)


1. SSH into the server 
2. Download iodine. (sudo apt-get install iodine for Ubuntu)
3. ```sh sudo iodined -c -r -f 10.0.0.1 -P <anypassword>  n.mydomain.com```

This run iodine daemon in server on 10.0.0.0 subnet with password <anypassword>. 
-f is to run iodine in foreground. 
-r is to skip raw UDP mode attempt. This is important. While I was testing this I was testing on internet uncensored network. It will try to connect with original public ip without -r. This is called raw mode in iodine. 
4. Verify. 
You can verify if your iodine setup works in iodine website. [Check it](https://code.kryo.se/iodine/check-it/). Paste your n.mydomain.com there and check. Successfully setup iodine server will response as below. 

![Iodine](/images/testing_iodine.PNG)

## Setting up client. 
For client I am using linux laptop to connect. Iodine provides binaries for multiple platforms including Android. I have tested both Android, Linux and Windows clients. I somehow can't able to connect from window client. Seems like its broken. But Android and Linux clients are ok to connect. 
[Android client](https://github.com/yvesf/andiodine) 
[Linux client](https://code.kryo.se/iodine)

1. Install iodine in machine. (sudo apt-get install iodine for Ubuntu)
2. Connect to iodine server. 
```sh iodine -I 50 -f -P <yourpassword> n.mydomain.com```

![Iodine](/images/iodine.jpg) 

Remember Server tunnel ip from the output. (10.0.0.1 in the picture)

It's ok if you don't get `SEGVFAIL` responses. Its normal not to get them. Strangely I am getting SEGVFAIL responses. Seems like my DNS server is timingout before the request completed. Don't worry if you get `SEGVFAIL` too. iodine will try to increase the timeout automatically.

Now you should be able to ping from both server and client. 

### Client 
1. ping server tunnel ip (10.0.0.1 in our example). 
You should be able to ping the server successfully.
2.  ```sh ip a``` and copy the ip of dns0 (10.0.0.2 in the picture) 
![DNS0](/images/dns0.jpg)


### Server 
1. ssh into server
2. ping client ip copied from dns0 
You should be able to ping the client successfully. 

Now we are connected!.

Now you can setup ipforwarding rules to pass the network route throught server tunnel (10.0.0.1). 
You can also setup simple proxies with ssh forwarding through server tunnel ip.

For example, you can easily setup a sock5 proxy with ssh port forwarding 

```sh ssh -DN 1337 10.0.0.1```

And test in curl. This will returns the `server public ip` 

```curl -x socks5h://127.0.0.1:1337 http://httpbin.org/ip```

You can also setup your own VPN through dns tunnel.  Now we have successfully setup slow internet connection over DNS bypassing firewalls and restrictions.
