# How to setup an AWS Client VPN

AWS Client VPN is a managed client-based VPN service that enables you to securely access your AWS resources and resources in your on-premises network. With Client VPN, you can access your resources from any location using an OpenVPN-based VPN client.

This was an assignment during [SAA Certification Bootcamp- E9 - Networking Services] (https://youtu.be/ES7inmRo2Is?t=7071)

This is Part I of the series, In this one, we will be focusing on **Access to a VPC**

Part II will be **Access to a peered VPC**

We are going to setup an AWS Client VPN in US East (N. Virginia), let's get started

![AWS Client VPN - Architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t9596005khtniwhl4yxf.png)

## Table of Contents
1. [Create a VPC](#vpc)
2. [Authentication](#auth)
    * [Generate Certificates](#certs)
    * [Upload Certificates to ACM](#upload)
3. [Create a Client VPN Endpoint](#vpnendpoint)
4. [Enable VPN connectivity for clients](#enable)
5. [Authorize clients to access a network](#authclients)
6. [(Optional) Enable access to additional networks](#optional)
7. [Download the Client VPN endpoint configuration file](#download)
8. [Connect to the Client VPN endpoint](#connect)
9. [Testing](#test)
10. [Client VPN scaling considerations](#scaling)
    * [Client CIDR range size](#cidr)
    * [Number of associated subnets](#number)
11. [Cost](#cost)
12. [Referrals](#referrals) 

<a name="vpc"></a>
### Step 1. Create a VPC
1. Navigate to `VPC Console` > `Create a VPC`
2. Enter Name as 'vpn-vpc' and IPV4 CIDR as `192.168.0.0/16`
   
![VPC](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g65y96p0pok3qd62xq5w.png)

3. Navigate to `Subnet` > Click `Create a Subnet` > Select `vpn-vpc` > Subnet name as `private-subnet` and IPV4 CIDR as `192.168.0.0/27`
   
![Subnet](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rz88yxij9991jlktek3z.png)

> Note: We will launch an EC2 instance into this subnet for a connectivity test at the end. 

---

<a name="auth"></a>
### Step 2. Authentication
Client VPN uses Authentication as the first point of entry into the AWS cloud. There are three ways to authenticate, we are going to use Mutual authenticaiton (certificate-based) 

* [Active Directory authentication (user-based)](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html#ad)
* [Mutual authentication (certificate-based)](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html#mutual)
* [Single sign-on (SAML-based federated authentication) (user-based)](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html#federated-authentication)

---

<a name="certs"></a>
#### Generate Certificates

```
git clone https://github.com/kasukur/clientvpn.git
cd clientvpn
```

```
chmod 755 generate_certs.sh
./generate_certs.sh
```
> Tip: Enter a `directory` name and then you could press ENTER at `Common Name` or provide a name

---

<a name="upload"></a>
#### Upload Certificates to ACM

You do not necessarily need to upload the client certificate to ACM. If the server and client certificates have been issued by the same Certificate Authority (CA), you can use the server certificate ARN for both server and client when you create the Client VPN endpoint. 

For this exercise, we are going to upload the server and the client certificates to ACM. Be sure to use the same Region in which you intended to create the client VPN endpoint.

**Server Certificate**
```
aws acm import-certificate --certificate fileb://server.crt --private-key fileb://server.key --certificate-chain fileb://ca.crt
```

**Client Certificate**
```
aws acm import-certificate --certificate fileb://client1.domain.tld.crt --private-key fileb://client1.domain.tld.key --certificate-chain fileb://ca.crt
```

---

<a name="vpnendpoint"></a>
### Step 3. Create a Client VPN endpoint

1. Navigate to `VPC Console` > `Client VPN Enpoints` > `Create Clinet VPN EndPoint`
2. Provide a `name` and `description` (optional) for the Client VPN endpoint
3. Enter a Client IPv4 CIDR as `10.0.0.0/22`

> Note: The IP address range cannot overlap with the target network or any of the routes that will be associated with the Client VPN endpoint. The client CIDR range must have a block size that is between /12 and /22 and not overlap with VPC CIDR or any other route in the route table. You cannot change the client CIDR after you create the Client VPN endpoint.

4. Select `Server certificate ARN`, `Client certificate ARN` and check `Use mutual authentication`
5. Select no for `Connection Logging` and `Client Connect Handler`
6. Select `Enable split-tunnel`
7. Leave the rest of the default settings, and choose `Create Client VPN Endpoint`.


![ClientVPNEndpoint](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hub2x426wqx06lv6b4cg.png)


![SplitTunnel](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rby3x930newghg2tiake.png)

The status of Client VPN endpoint will be in `pending-associate` at this stage.

---

<a name="enable"></a>
### Step 4. Enable VPN connectivity for clients

1. Navigate to `VPC Console` > `Client VPN Enpoints` > Choose `Clinet VPN EndPoint` > Click `Associations` > Click `Asscociate`
2. Select the VPC (same VPC as in Step 1) and a subnet
3. Click on `Associate`

> Note: If authorization rules allow it, one subnet association is enough for clients to access a VPC's entire network. You can associate additional subnets to provide high availability in case one of the Availability Zones goes down.


![VPNEndpoint-Association](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/os3zaiqoopg7zr3bh16g.png)

The following are created when we associate the Client VPN Enpoint with the first subnet
- The state of the Client VPN endpoint changes to available. Clients can now establish a VPN connection, but they cannot access any resources in the VPC until you add the authorization rules.


![Associated](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7ffc7gkmeq4s46ak0wrw.png)

- The local route of the VPC is automatically added to the Client VPN endpoint route table.


![Route](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0ho6oun23vovmc69mjuj.png)

- The VPC's default security group is automatically applied for the subnet association. 


![Security Group](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3xfqsvoc361v6y8jrzxk.png)

---

<a name="authclients"></a>
### Step 5. Authorize clients to access a network
1. Navigate to `VPC Console` > `Client VPN Enpoints` > Choose `Clinet VPN EndPoint` > Click `Authorization` > Click `Authorize Ingress`
2. Enter `192.168.0.0/16` for `Destination network to enable`, `Allow access to all users` for `Grant access to` and Description as `VPC-through-VPNEndPoint`
3. Click `Add authorization rule`

![Add authorization rule](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4q3jdintllcob0t3rsyq.png)

---

<a name="optional"></a>
### Step 6. (Optional) Enable access to additional networks
You can enable access to additional networks connected to the VPC, such as AWS services, peered VPCs, and on-premises networks. For each additional network, you must add a route to the network and configure an authorization rule to give clients access.
More information at Step 5 of [Getting started with Client VPN](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/cvpn-getting-started.html#cvpn-getting-started-endpoint)

---

<a name="download"></a>
### Step 7. Download the Client VPN endpoint configuration file
1. Navigate to `VPC Console` > `Client VPN Enpoints` > Click `Download Client Configuration`
2. Open Client VPN endpoint configuraiton file and add the contents of the client certificate between <cert></cert> tags and the contents of the private key between <key></key> tags.
```
<cert>
Contents of client certificate (.crt) file, which is client1.domain.tld.crt under the same direcroty when the server and client certificates are located
</cert>

<key>
Contents of private key (.key) file, which is client1.domain.tld.crt
</key>
```
3. Prepend a random string to the Client VPN endpoint DNS name and add it to Client VPN endpoint configuraiton file as shown below.

```
remote srivpc.cvpn-endpoint-072a9dd48228b525b.prod.clientvpn.us-east-1.amazonaws.com 443 
```
4. Save the file.
5. You could distribute the Client VPN endpoint configuration file to your clients if required.

[Sample Client VPN endpoint configuraiton file](https://github.com/kasukur/clientvpn/blob/main/downloaded-client-config-nv.ovpn)

---

<a name="connect"></a>
### Step 8: Connect to the Client VPN Endpoint

> Note: This is for Mac OS

1. Download latest [VPN client application] (https://docs.aws.amazon.com/vpn/latest/clientvpn-user/client-vpn-connect-macos.html) 
2. Open the AWS VPN Client app.
3. Choose File, Manage Profiles.
![File Manage Profiles](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j8uqnatfx93do7begf5y.png)
4. Enter `Display Name`, Select `VPN Configuration File` and click `VPN Configuration File`
![Add Profile](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/62kneb6zhoz81o8skzto.png)
5. Click Connect.

---

<a name="test"></a>
### Step 9: Testing
1. When the connectivity is established, you can see OpenVPN Statistics by clicking on `Connection` > `Show Details`
![OpenVPN Stats](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/75j14zw2lff1bclvvl42.png)
2. You could also check under `Connections` tab under `Client VPN Endpoint`
![Connection Test](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/34wd1pek5jpa5yloyfxc.png)
3. We can also launch an EC2 instance in the private subnet created in Step 1 and connect to it.


![EC2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/va265km4opiv431xutt2.png)

![EC2 Test](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tuudt41yperpiqndchk4.png)

---

<a name="scaling"></a>
### Client VPN scaling considerations

[Source] (https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/scaling-considerations.html)

The maximum number of concurrent VPN connections depends on the following

---

<a name="cidr"></a>
#### Client CIDR range size

:point_right: _Points to note:_
- When we create a Client VPN endpoint, we must specify a client CIDR range, which is an IPv4 CIDR block between a /12 and /22 netmask. 
- Each VPN connection to the Client VPN endpoint is assigned a unique IP address from the client CIDR range. 
- A portion of the addresses in the client CIDR range are also used to support the availability model of the Client VPN endpoint, and cannot be assigned to clients. 
- We cannot change the client CIDR range after you create the Client VPN endpoint.

---

<a name="number"></a>
#### Number of associated subnets

:point_right: _Points to note:_
- When we associate a subnet with a Client VPN endpoint, we enable users to establish VPN sessions to the Client VPN endpoint. We can associate multiple subnets with a Client VPN endpoint for high availability, and to enable additional connection capacity.
- We cannot associate multiple subnets from the same Availability Zone with a Client VPN endpoint. Therefore, the number of subnet associations also depends on the number of Availability Zones that are available in an AWS Region.
- For example, if you expect to support 4,000 VPN connections to your Client VPN endpoint, specify a minimum client CIDR range size of /19 (8,192 IP addresses), and associate at least 2 subnets with the Client VPN endpoint.
- If you’re unsure what the number of expected VPN connections is for your Client VPN endpoint, we recommend that you specify a size /16 CIDR block or larger.

---

<a name="cost"></a>
### Cost
For US East (N. Virginia)
- AWS Client VPN endpoint association	$0.10 per hour
- AWS Client VPN connection	$0.05 per hour
- There is a cost for an `Asscociated VPN Endpoint` even when not in use, so `Disassociate` when not in use. 

---

<a name="referrals"></a>
### Referrals
- [Authentication](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html#mutual)
- [Getting started with Client VPN](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/cvpn-getting-started.html)


---
