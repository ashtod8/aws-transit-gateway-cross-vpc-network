# aws-transit-gateway-cross-vpc-network
A secure, cost-optimized cloud network implementation connecting two isolated VPCs (10.1.0.0/16 and 10.2.0.0/16) via a standalone AWS Transit Gateway. Features custom Network ACLs, security group ICMP validation rules, and an EC2 Instance Connect Endpoint (EICE) for secure, private management without the overhead of a NAT Gateway.


## Step-by-Step Implementation Guide

### Step 1: Create the Virtual Private Clouds (VPCs)
1. Navigate to the **VPC Console** and create two separate VPCs with non-overlapping CIDR blocks:
   * **`VPC-Internal`**: CIDR block `10.1.0.0/16`
   * **`VPC-Dev`**: CIDR block `10.2.0.0/16`
2. Within each VPC, provision both a public subnet and a private subnet across different Availability Zones (e.g., `ap-south-1a` and `ap-south-1b`).
*Verification:* Check that both VPCs show an `Available` state with their respective IPv4 CIDR allocations correctly assigned.

### Step 2: Configure Internet Gateways and Route Tables
1. Create an **Internet Gateway (IGW)** for each VPC and attach them to their respective VPCs (`Internal-IGW` and `DEV-IGW`).
2. Create public route tables and add a default route (`0.0.0.0/0`) pointing to the Internet Gateway. Associate your public subnets with these route tables.
3. Keep private route tables isolated with local routing only (`10.1.0.0/16` or `10.2.0.0/16` set to `local`).
*Verification:* Confirm that public subnets display an active route to the Internet Gateway.

### Step 3: Deploy the Transit Gateway (TGW)
1. Go to **Transit Gateways** in the VPC Console and click **Create transit gateway**.
2. Name your TGW (e.g., `Core-Transit-Gateway`), leave default options enabled, and create it.
3. Once available, create **Transit Gateway Attachments** for both VPCs:
   * **TGW-1**: Attach `VPC-Internal` and select its private subnets.
   * **TGW-2**: Attach `VPC-Dev` and select its private subnets.
*Verification:* Ensure both attachments transition from `pending` to `Available`.

### Step 4: Configure Cross-VPC Routing via TGW Route Tables
1. Update your private subnet route tables in both VPCs to allow cross-network traffic through the Transit Gateway:
   * In **Internal-Private-RT**, add a route where Destination is `10.2.0.0/16` and Target is your Transit Gateway (`tgw-xxxxxx`).
   * In **DEV-Private-RT**, add a route where Destination is `10.1.0.0/16` and Target is your Transit Gateway (`tgw-xxxxxx`).
*Verification:* Verify that the target entries in both private route tables point directly to your Transit Gateway ID.

### Step 5: Set Up Security Controls (NACLs & Security Groups)
1. **Network ACLs (Stateless)**: Modify custom Network ACLs on your private subnets to explicitly allow ICMP traffic:
   * **Inbound Rule**: Allow `All ICMP - IPv4` from source `10.1.0.0/16` (for Dev) or `10.2.0.0/16` (for Internal).
   * **Outbound Rule**: Allow `All ICMP - IPv4` to destination `10.1.0.0/16` or `10.2.0.0/16`.
2. **Security Groups (Stateful)**:
   * Update the security group attached to `EC2-Internal` to include an inbound rule allowing `All ICMP - IPv4` from the `10.2.0.0/16` CIDR.
   * Update the security group attached to `EC2-Dev` to include an inbound rule allowing `All ICMP - IPv4` from the `10.1.0.0/16` CIDR.
*Verification:* Inspect the rules tables to ensure ICMP protocol rules are active and correctly scoped to the peer CIDRs.

### Step 6: Deploy EC2 Instance Connect Endpoint (EICE) for Secure Access
1. In your public subnet within `VPC-Internal`, create an **EC2 Instance Connect Endpoint** to allow secure terminal access without assigning public IP addresses to your private instances.
2. Launch a private test instance (`EC2-Internal`) in the private subnet of `VPC-Internal` and another (`EC2-Dev`) in `VPC-Dev`.
*Verification:* Connect to `EC2-Internal` via the AWS Console using the EICE terminal option.

### Step 7: Validate Cross-VPC Connectivity
1. From your browser-based terminal session on `EC2-Internal`, execute a ping test targeting the private IP address of `EC2-Dev`:
   ```bash
   ping <Private-IP-of-EC2-Dev>
