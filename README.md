# Scalable & High Availability Streaming Engine Deployment on AWS

## Overview
This setup deploys a scalable, clustered Ant Media Server on AWS for real-time, ultra-low-latency live streaming. It provision a VPC with public/private subnets, an Application Load Balancer (ALB) for WebRTC/HTTPS traffic, a Network Load Balancer for RTMP/SRT, Auto Scaling Groups for Ant Media Origin (ingest/transcode) and Edge (delivery) nodes, and a MongoDB backend for clustering. Ant Media’s WebRTC-based streaming delivers latency around ~0.5 seconds, ideal for interactive EdTech applications. Origin nodes ingest and process streams, edges fetch from origins, and clients always connect via the load balancers for high availability.

----

## Prerequisites
- **AWS Account**: Administrator access to create VPCs, subnets, EC2 instances, Load Balancers, IAM users/roles, etc.
- **Ant Media Enterpris Edition Subscription**: Enterprise edition subscription or an AMI that includes the license (required for clustering and advanced features).
- **SSH Key Pair**: An EC2 key pair name for SSH access to instances. This is passed as the `KeyName` parameter.
- **ACM Certificate ARN**: A pre-issued ACM certificate in the target AWS Region for your domain (to enable HTTPS on the ALB). Provide its ARN via the `LoadBalancerCertificateArn parameter`.
- **Custom Domain**: A domain managed in Route 53 (for cleaner URLs). You can point a Route 53 A/alias record to the ALB after deployment.

----

## Deployment Steps
1. **Launch CloudFormation**: In the AWS Console, navigate to CloudFormation and click Create stack. Upload the provided yaml file or specify its S3 URL.
2. **Configure Parameters**: Fill in required parameters, for example:
   - **VpcCidrBlock** (e.g. `10.0.0.0/16`) – IP range for the new VPC.
   - **OriginCidrBlock**, **EdgeCidrBlock** (subnet CIDRs, e.g. `10.0.1.0/24`, `10.0.2.0/24`).
   - **Instance Types**: Choose `OriginInstanceType` and `EdgeInstanceType` (for example `c5.xlarge`).
   - **MongoDBInstanceType** (e.g. c5.large).
   - **KeyName**: Your SSH key pair name.
   - **SSHLocation**: Your IP/CIDR to allow SSH (default is `0.0.0.0/0`; it’s safer to restrict to your IP).
   - **Email**: A contact email for auto-scaling notifications.
   - **AntMediaOriginCapacity** and **AntMediaEdgeCapacity**: Initial number of Origin/Edge nodes (e.g. 1 and 2).
   - **LoadBalancerCertificateArn**: Paste your ACM certificate ARN.
   - **DiskSize**: (optional) root volume size in GB.
3. **Launch Stack**: Submit the stack. CloudFormation will provision all resources (VPC, subnets, Internet Gateway, Route Tables, Security Groups, Load Balancers, AutoScaling Groups, EC2 instances, etc.).
4. **Wait for Completion**: The stack may take several minutes. Monitor the Events in the CFN console until it reaches `CREATE_COMPLETE`.
5. **Retrieve Endpoints**: Once complete, go to the **Outputs** tab. Note the ALB DNS name or provided URLs:
   - **Dashboard (HTTPS)**: `https://<ALB-DNS>` – Ant Media dashboard UI.
   - **Origin & Edge URLs**: HTTPS endpoints for origin and edge clusters (port 5443 for WebRTC).
   - **RTMP URL**: `rtmp://<RTMP-LB-DNS>`.
   - **HTTP endpoints** (for Redirection/Configuration, ports 5080/5081) are also provided.
   *(All endpoints use the ALB DNS by default. If using a custom domain, the same domain will map to the ALB once DNS is configured.)*

----

## Post-Deployment Configuration
- **Log into Dashboard**: Open the Dashboard HTTPS URL (from Outputs) in a browser. Log in with the default Ant Media credentials or your license’s configured credentials.
- **Create Live Stream**: In the Applications panel, go to **live** (or your app), then Applications Settings to ensure **WebRTC, HLS, and S3 recording** are enabled. Then create a new stream (give it an ID/name).
- **Verify MongoDB Cluster Registration**: In the Ant Media **Cluster** page, you should see all Origin and Edge nodes listed (by private IP). Each node auto-registers itself in MongoDB on startup. All nodes (even inactive ones) appear; you can delete stale entries if needed. This confirms the cluster is connected to MongoDB.
- **Check Nodes**: In EC2 console, verify your Origin and Edge instances are running in the correct subnets, with Name tags `Antmedia-Origin` and `Antmedia-Edge`. The MongoDB instance (tagged Antmedia-MongoDB) should also be running.
----

## Publishing Streams
### RTMP via OBS:
Install OBS Studio. In OBS Settings → Stream, set Service to Custom. For the server URL, use the RTMP endpoint without a port, e.g.:
```js
rtmp://<ALB-DNS-or-domain>/live
```
*(Use your live or chosen application name.) Set the Stream Key to your stream ID (e.g. `mystream`). Start streaming. Ant Media will ingest it. Remember: do not include :`1935` in the URL as OBS auto-uses 1935.*

### WebRTC Publishing:
Use Ant Media’s built-in HTML5 publisher. In a browser, navigate to:
```js
https://<ALB-DNS-or-domain>:4444/LiveApp/index.html
```
*This is the sample page in `/usr/local/antmedia/webapps/live/index.html`.) Allow camera/mic access, click Start and then publish. This uses WebRTC for ultra-low-latency ingest.*

----

## Viewing Streams
### WebRTC Playback:
For lowest latency, use the sample player at:
```js
https://<ALB-DNS-or-domain>:4444/live/play.html?id=<STREAM_ID>
```
*(Located at /usr/local/antmedia/webapps/LiveApp/player.html.) It will play via WebRTC by default. Ensure your stream exists and is active.*

### HLS Playback (video.js):
Ant Media supports HLS output. The HLS playlist URL is
```js
https://<ALB-DNS-or-domain>:4444/live/streams/<STREAM_ID>.m3u8
```
or
```js
https://<ALB-DNS-or-domain>:4444/live/play.html?id=<STREAM_ID>&playOrder=hls
```

To play this in a browser, use a compatible player like video.js or [hls.js]. For example, embed a Video.js player on a webpage:
```HTML
<video id="hls-video" class="video-js vjs-default-skin" controls preload="auto" width="640" height="360">
  <source src="https://<ALB-DNS-or-domain>:4444/live/streams/<STREAM_ID>/playlist.m3u8" type="application/x-mpegURL">
</video>
<script src="https://vjs.zencdn.net/7.x/video.min.js"></script>
```
The Ant Media Web Player (open-source) also uses Video.js under the hood and can be embedded via their React/JS SDK or using the supplied `play.html`.

----

## S3 Recording Setup

1. **Create IAM User**: In the AWS IAM console, create a new user with *Programmatic access*. Attach a policy with at least **s3:PutObject** and **s3:PutObjectAcl** (for example, the managed policy `AmazonS3FullAccess` for testing). Save the Access Key ID and Secret Key.
2. **Create S3 Bucket**: In S3, make a bucket (e.g. `recordings`). Note its name and Region. Configure `CORS` if you plan to play recordings via a web player (see AWS docs).
3. **Configure Ant Media**: In the dashboard, go to **Applications → live → Settings** (under S3 Recording). Enter the Access Key, Secret Key, S3 bucket name, and Region. Enable “Record Live Stream as MP4” and “Enable S3 Recording”. Save settings.
4. **Verify**: Start a stream and stop it to finalize recording. The resulting MP4 should appear in the S3 bucket within a few minutes. You can also play them via the S3 URL or through Ant Media’s VOD player.

----

## Domain Setup
To use a friendly domain (e.g. `class.example.com`) instead of the ALB DNS:
### ACM Certificate:
Ensure your domain’s certificate is imported or issued in ACM in the *same Region* as the ALB. Use DNS validation via Route 53 for automatic renewal.
### Route 53 Record:
In the Route 53 hosted zone for your domain, create the record. Set the alias target to the ALB’s DNS (find it in the EC2 or ELB console). This routes stream.example.com to the ALB. For root domains, alias is required (CNAME is not allowed at the root)
### SSL on ALB:
In the load balancer listeners, ensure there is an HTTPS (443) listener using your ACM certificate ARN (this is already configured by the template if you passed `LoadBalancerCertificateArn`).
- **After DNS propagates**, accessing https://stream.example.com will reach the Ant Media dashboard or player via the ALB.

----

## Security Best Practices
- **Lock Down SSH**: Edit the stack or Security Group to restrict SSH (port 22) access to known IPs. The SSHLocation parameter defaults to open (0.0.0.0/0). For production, set it to your office/home IP/CIDR.

- **Use IAM Roles**: Instead of embedding AWS keys, consider using IAM Roles for EC2 (if adding AWS APIs). At minimum, do not hardcode credentials. The S3 recording uses keys temporarily, but they are stored in Ant Media settings (local server).

- **Enforce HTTPS**: Ensure all client-facing traffic uses HTTPS (443) or secure WebSockets. The ALB should redirect HTTP (5080) to HTTPS or simply not be exposed. In Ant Media Application Settings, enable SSL on 5443, and disable non-SSL ports if not needed.

- **Cluster Network Security**: The template creates security groups. For internal communication, note Ant Media cluster uses port 5000/tcp between nodes; this should only be open inside the VPC. The template confines most ports to the VPC subnets.

- **Least Privilege**: Attach the minimal IAM policy to the S3 user (e.g. specific bucket access only). Remove any unused AWS resources after testing.

----

# Troubleshooting
- **Stack Launch Fails**: Check CloudFormation Events for missing resources or permission errors. Ensure all parameters (especially Certificate ARN and KeyName) are correct.

- **ALB Not Forwarding Traffic**: Verify ALB listeners (HTTPS on 443, HTTP on 80 or 5080) and target groups. The stack sets an ALB with HTTPS listener (port 443) forwarding to both Origin and Edge groups. Ensure the certificate is valid and listener rules are correct. Use the ALB Target Groups health checks to see if instances are healthy.

- **Cannot SSH**: Confirm your SSH key pair name matches an existing key. Ensure the SSHLocation CIDR includes your IP.

- **AMS Dashboard Unreachable**: Check that your domain/ALB listener is on 443 and certificate is attached. If using a custom domain, ensure Route 53 alias is correct and has propagated. Try accessing the ALB DNS directly to rule out DNS issues.
  
- **Streams Not Registered**: If Edge nodes report “no origin” or nothing plays, check that the MongoDB instance is running and accessible by the nodes. In the InstanceSecurityGroup rules, port 27017 (MongoDB) should be open between the nodes. Also check that each node’s user data ran successfully (look at /usr/local/antmedia/log on instances).

- **No Video on Playback**: Ensure your publisher is actually sending data. In Ant Media Live Streams table, the status should be broadcasting. For WebRTC, check browser console for errors. For HLS, confirm the .m3u8 playlist and segments exist (you can curl them).

- **S3 Upload Failures**: Verify the IAM user’s permissions and that the bucket name/region are correct. Check the Ant Media logs (usually /usr/local/antmedia/log/) for S3 errors. If using a bucket with special requirements (like encryption or requester-pays), ensure those are supported.
