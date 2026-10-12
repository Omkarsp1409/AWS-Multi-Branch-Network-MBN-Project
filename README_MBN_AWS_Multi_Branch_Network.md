# AWS Multi-Branch Network (MBN) Project

> A hands-on AWS networking project connecting branch VPCs to a
> centralized application VPC using AWS Transit Gateway, with controlled
> routing and private-subnet internet egress.

## Overview

The Multi-Branch Network (MBN) project demonstrates a centralized
network design for three branch environments---Pune, Mumbai, and
Delhi---and a shared application environment hosted in AWS.

The design uses separate VPCs, AWS Transit Gateway attachments,
dedicated Transit Gateway route tables for each branch, Amazon EC2,
Apache HTTP Server, security groups, network ACLs, an Internet Gateway,
and a NAT Gateway.

**Region:** Asia Pacific (Mumbai) --- `ap-south-1`

## Architecture

``` text
 Pune Branch VPC                 Mumbai Branch VPC               Delhi Branch VPC
 10.1.0.0/16                     10.2.0.0/16                     10.3.0.0/16
       |                                |                               |
       +---------------- Transit Gateway (MBN-Central-TGW) -------------+
                                      |
                              Application VPC
                                10.10.0.0/16
                                      |
                          Application EC2: 10.10.11.94
                            Apache HTTP Server :80
                                      |
                         Private subnet -> NAT Gateway
                                      |
                              Internet Gateway
                                      |
                                   Internet
```

> Diagram is a logical overview. Exact AZ placement and every route are
> defined in the AWS console configuration.

## Network Address Plan

  ------------------------------------------------------------------------
  Environment       VPC CIDR          Workload subnet    Purpose
  ----------------- ----------------- ------------------ -----------------
  Pune branch       `10.1.0.0/16`     `10.1.1.0/24`      Branch workload

  Mumbai branch     `10.2.0.0/16`     `10.2.1.0/24`      Branch workload

  Delhi branch      `10.3.0.0/16`     `10.3.1.0/24`      Branch workload

  Application       `10.10.0.0/16`    `10.10.11.0/24`    Central
                                                         application

  Application       Within            `10.10.1.0/24`     Public NAT
  public subnet     `10.10.0.0/16`                       Gateway subnet

  Application TGW   Within            `10.10.255.0/28`   Transit Gateway
  subnet            `10.10.0.0/16`                       attachment

  Mumbai TGW subnet Within            `10.2.255.0/28`    Transit Gateway
                    `10.2.0.0/16`                        attachment
  ------------------------------------------------------------------------

The Pune and Delhi workload subnet ranges shown above are the intended
plan; confirm exact subnet names/CIDRs against the final AWS console
before treating them as an authoritative inventory.

## Main AWS Components

-   **Amazon VPC:** Isolated networks for each branch and the central
    application environment.
-   **AWS Transit Gateway:** Central hub connecting the VPCs.
-   **Transit Gateway attachments:** Connect each VPC to the Transit
    Gateway.
-   **Transit Gateway route tables:** Dedicated branch route tables are
    used to control which destinations each branch can reach.
-   **Amazon EC2:** Runs the centralized application server.
-   **Apache HTTP Server:** Serves the sample web page over HTTP.
-   **Internet Gateway (IGW):** Provides internet connectivity for
    resources in public subnets with appropriate routes and addressing.
-   **NAT Gateway:** Enables outbound internet access from private
    subnets, for example to download operating-system packages.
-   **Security Groups and Network ACLs:** Control traffic at the
    network-interface and subnet levels.

## Transit Gateway Routing Design

The intended routing model is:

-   `MBN-Pune-TGW-RT`: route to `10.10.0.0/16` through the Application
    attachment.
-   `MBN-Mumbai-TGW-RT`: route to `10.10.0.0/16` through the Application
    attachment.
-   `MBN-Delhi-TGW-RT`: route to `10.10.0.0/16` through the Application
    attachment.
-   `MBN-App-TGW-RT`: routes to branch CIDRs `10.1.0.0/16`,
    `10.2.0.0/16`, and `10.3.0.0/16` through their respective
    attachments.

The Application VPC subnet route table (`MBN-Private-RT`) contains
routes for the branch CIDRs through the Transit Gateway. It also needs a
default route to the NAT Gateway for internet-bound traffic while
preserving the more-specific branch routes.

### Branch isolation note

Separate Transit Gateway route tables are intended to prevent direct
branch-to-branch routing. Treat isolation as **not fully verified until
explicit cross-branch connectivity tests have been performed**. Also
review any alternate routes, peering, VPNs, or other paths before making
a security guarantee.

## Application Hosting

The centralized application EC2 private IP is:

``` text
10.10.11.94
```

Apache HTTP Server serves the web page from:

``` text
/var/www/html/index.html
```

Common setup commands on Amazon Linux 2023:

``` bash
sudo dnf install httpd -y
sudo systemctl enable --now httpd
sudo systemctl is-active httpd
```

Example local test:

``` bash
curl http://localhost
```

Example branch tests:

``` bash
curl -I http://10.10.11.94
```

Run the branch test from each branch EC2. The Application security group
should allow TCP port 80 from only the intended branch CIDRs, rather
than from the entire internet.

## Internet Egress for Private Instances

The package-download path is:

``` text
Private Application EC2
  -> Application private route table (0.0.0.0/0)
  -> NAT Gateway in public subnet
  -> Internet Gateway
  -> Internet
```

The public subnet route table must contain
`0.0.0.0/0 -> Internet Gateway`. The private subnet route table must
contain `0.0.0.0/0 -> NAT Gateway`. Keep the more-specific routes to the
branch CIDRs through the Transit Gateway.

**Cost note:** NAT Gateways and Transit Gateways can incur hourly and
data-processing charges. Review current AWS pricing and delete resources
that are no longer needed.

## Validation Results

Based on the hands-on tests reported during implementation:

  -----------------------------------------------------------------------
  Test                    Reported result         Notes
  ----------------------- ----------------------- -----------------------
  Pune EC2 to Application Successful              ICMP connectivity was
  EC2                                             established

  Mumbai EC2 to           Successful              Connectivity was
  Application EC2                                 established after
                                                  troubleshooting

  Pune to hosted HTTP     Successful              `curl`/HTTP access
  application                                     reported successful

  Mumbai to hosted HTTP   Successful              `curl`/HTTP access
  application                                     reported successful

  Delhi to Application    Not tested              No confirmed end-to-end
  EC2                                             result

  Branch-to-branch        Requires explicit       Do not infer isolation
  isolation               validation              solely from application
                                                  connectivity

  NAT internet/package    Configured for package  Validate with an
  access                  downloads               outbound HTTPS/package
                                                  test if not already
                                                  recorded
  -----------------------------------------------------------------------

## Troubleshooting Approach

When a branch cannot reach the application, troubleshoot in a controlled
order:

1.  Confirm source and destination private IP addresses.
2.  Check the source subnet's VPC route to the Application CIDR through
    the Transit Gateway.
3.  Confirm the source attachment is associated with the expected
    Transit Gateway route table.
4.  Confirm the route table has the destination route to the Application
    attachment.
5.  Confirm the Application attachment's associated route table has a
    return route to the branch CIDR.
6.  Review security groups and network ACLs in both directions.
7.  Use `ping`, `curl`, `tcpdump`, or VPC Flow Logs as appropriate to
    narrow down where packets are lost.
8.  Change one setting at a time and retest.

## Security Considerations

-   Restrict SSH access to trusted administrative sources or use a
    managed access method.
-   Allow HTTP only from required branch CIDRs if the application is
    internal-only.
-   Do not expose the application to `0.0.0.0/0` unless public access is
    explicitly required and protected.
-   Verify both directions of stateful/stateless traffic handling.
-   Perform explicit branch-to-branch tests before claiming isolation.
-   Consider CloudWatch monitoring, VPC Flow Logs, patching, backups,
    and recovery testing before production use.

## Known Limitations / Future Improvements

-   Test Delhi-to-application connectivity end to end.
-   Perform explicit Pune-to-Mumbai, Pune-to-Delhi, and Mumbai-to-Delhi
    isolation tests.
-   Review high availability across Availability Zones and failure
    recovery.
-   Configure monitoring and alerts.
-   Review Transit Gateway and NAT Gateway costs.
-   Consider infrastructure as code using Terraform or AWS
    CloudFormation.
-   Add a verified architecture diagram and redacted AWS console
    screenshots to the repository.

## Repository Evidence

Suggested repository structure:

``` text
mbn-aws-multi-branch-network/
├── README.md
├── docs/
│   ├── architecture.png
│   └── configuration-evidence/
│       ├── vpc-overview.png
│       ├── transit-gateway-attachments.png
│       ├── transit-gateway-routes.png
│       └── application-test-results.png
└── scripts/
    └── (optional scripts, if created)
```

Before publishing screenshots, redact account IDs, public IP addresses,
resource identifiers you consider sensitive, and any credentials or
secrets. Never commit private keys, access keys, passwords, or session
tokens.

## What I Learned

-   Designing VPC CIDR ranges and subnets for multiple environments.
-   Connecting VPCs using AWS Transit Gateway.
-   Using Transit Gateway route-table associations and routes to control
    network paths.
-   Hosting a centralized application with EC2 and Apache.
-   Providing private-subnet outbound access with NAT Gateway and
    Internet Gateway.
-   Troubleshooting connectivity using routing, security rules, and
    packet-level diagnostics.
-   Documenting implementation evidence and separating tested outcomes
    from assumptions.

------------------------------------------------------------------------

**Project status:** Core networking and centralized application access
from Pune and Mumbai were reported as working. Delhi connectivity,
explicit branch-isolation testing, and production-readiness items remain
to be verified before describing the architecture as production-ready.
