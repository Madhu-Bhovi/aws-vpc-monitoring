# Custom VPC with Public and Private Subnets and CloudWatch Monitoring

A custom AWS network built by hand in the console: one VPC, a public and a private subnet, an internet gateway, separate route tables, and security groups with least-privilege rules. A web server runs in the public subnet, a second server sits in the private subnet to prove isolation, and a CloudWatch alarm emails me through SNS when the web server's CPU is high.

## What I built

| Component        | Details                                                                                                                          |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| VPC              | `project2-vpc`, 10.0.0.0/16, ap-south-1 (Mumbai), DNS hostnames enabled                                                          |
| Public subnet    | 10.0.1.0/24 in ap-south-1a, auto-assigns public IPs                                                                              |
| Private subnet   | 10.0.2.0/24 in ap-south-1a                                                                                                       |
| Internet gateway | `project2-igw`, attached to the VPC                                                                                              |
| Route tables     | Public: local + `0.0.0.0/0` to the gateway. Private: local only                                                                  |
| Security groups  | `project2-web-sg` and `project2-private-sg` (rules below)                                                                        |
| Web server       | `project2-web`, t3.micro, Ubuntu 24.04, public IP, runs a Flask to-do app behind Nginx and Gunicorn                              |
| Private server   | `project2-private`, t3.micro, Ubuntu 24.04, no public IP                                                                         |
| Monitoring       | CloudWatch alarm on `CPUUtilization` (50% or more, 5-minute average) sending to the SNS topic `project2-alerts`, which emails me |

### Security group rules (inbound)

| Security group        | Protocol / Port       | Source            | Purpose                              |
| --------------------- | --------------------- | ----------------- | ------------------------------------ |
| `project2-web-sg`     | HTTP (80)             | `0.0.0.0/0`       | Public access to the Flask to-do app |
| `project2-web-sg`     | SSH (22)              | My IP             | Admin access and jump host entry     |
| `project2-private-sg` | SSH (22), ICMP (ping) | `project2-web-sg` | Access only from the web server      |

The to-do app is the one from [project 1](https://www.google.com/search?q=https://github.com/Madhu-Bhovi/aws-todo-ec2), deployed with the Nginx and systemd files saved in that repo. The app is only the workload here, and the focus of this project is the network and the monitoring.

## Design decisions

| Decision                                                 | Reason                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| /16 VPC with /24 subnets                                 | Plenty of room to add subnets later. Each /24 has 251 usable addresses, since AWS reserves 5.                            |
| Public versus private decided by route table             | A subnet is public only because its route table has a route to the internet gateway.                                     |
| Web group: HTTP 80 from anywhere, SSH 22 from my IP only | The website must be public, and administration should not be.                                                            |
| Private group: SSH and ping only from `project2-web-sg`  | Using a security group as the source means "only members of the web group", and it keeps working if IP addresses change. |
| No public IP and no internet route on the private server | The internet cannot reach it, and it cannot start outbound connections.                                                  |
| No NAT Gateway                                           | It bills every hour. The trade-off is that the private server cannot download updates.                                   |
| No RDS database                                          | Cost. The private server only demonstrates network isolation.                                                            |
| SSH to the private server through the web server         | A jump host. The key stays on my laptop and is never copied to a server.                                                 |
| Detailed monitoring off                                  | EC2 reports every 5 minutes by default, and 1-minute metrics cost extra.                                                 |
| Alarm threshold of 50%                                   | Low enough to trigger on purpose for a demo. Production alarms would use a higher value.                                 |

## What I tested

| Test                                                                             | Result                                                                     |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Web page loads from the internet through the internet gateway                    | Passed                                                                     |
| SSH to the private server from my laptop using its private address               | Blocked: connection timed out after 8 seconds                              |
| SSH to the private server through the web server                                 | Passed, after fixing my SSH config (see problems)                          |
| Ping the private server from the web server                                      | Passed: 3/3 packets received, 0% packet loss (0.198 ms avg)                |
| Private server reaches the internet (`curl https://example.com`, `ping 8.8.8.8`) | Blocked: `curl` timed out after 5 seconds and ping showed 100% packet loss |
| CPU alarm: `stress-ng` on 2 cores for about 5 minutes                            | Passed: the alarm fired and the alert email arrived at 12:36 IST           |

Command outputs are in [docs/test_output.txt](https://www.google.com/search?q=docs/test_output.txt).

## Screenshots

**Network**

**Security groups**

**Servers and app**

**Monitoring**

## Problems I ran into

| Problem                                                                                          | Cause                                                                                                                                                                                                                       | Fix                                                                                                                                                |
| ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Web page said "connection refused", but SSH worked                                               | Nginx was not installed. The user data script had not installed it, and the saved script text looked cut off at the end, so it may have been truncated when pasted into the launch form. I did not confirm the exact cause. | Installed Nginx by hand. A refused connection means the path works but nothing is listening, while a timeout means something is blocking the path. |
| `ssh -J` with `-i` returned "Permission denied (publickey)"                                      | Unconfirmed. Command-line options like `-i` are documented to apply to the destination, not to the jump host.                                                                                                               | Moved to a `ProxyJump` entry in `~/.ssh/config`.                                                                                                   |
| `ssh project2-private` returned "Could not resolve hostname" even though the config looked right | The config file had Windows (CRLF) line endings, shown as `^M` by `cat -A`, so the `Host` names never matched.                                                                                                              | Removed them with `sed -i 's/\r$//'` and set the editor to LF.                                                                                     |

## Cost

- Two t3.micro instances, each running from about 11:50 am IST until I stopped them at about 4 pm IST the same day. That is roughly 4 hours each, or about 8 instance-hours, plus a public IPv4 address on the web server while it ran.

- The VPC, subnets, route tables, internet gateway, security groups, SNS topic, and alarm cost nothing at this scale.

- Billing page total when I checked: \$2.19. That figure may not include these two servers yet, since billing data can lag by up to a day. \[update after the next check\]

- Resources were stopped or deleted between sessions, and Budget alerts at \$5 and $1 give early warning.

## Limitations and next steps

- **Single Availability Zone.** If it fails, both servers go down. A production design uses at least two.

- **No NAT Gateway**, so the private server cannot reach the internet.

- **HTTP only**, with no encryption in transit.

- **Built by hand in the console.** Next, in project 3, I will define the same network in Terraform so it can be recreated with one command.

- **Only a CPU alarm.** Memory and disk space need the CloudWatch agent, which I have not installed.

## Cleanup

- \[x\] Terminated both instances

- \[x\] Deleted the CloudWatch alarm and the SNS topic

- \[x\] Deleted the security groups, route tables, subnets, internet gateway, and VPC (in that order)

- \[x\] Deleted the key pair `project2-key`

- \[ \] Checked Billing, Bills the next day

## Project structure

```
aws-vpc-monitoring/
├── README.md
├── .gitignore
└── docs/
    ├── architecture.png
    ├── test_output.txt
    └── [screenshots]

```
