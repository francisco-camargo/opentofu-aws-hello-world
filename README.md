# OpenTofu Hello World on AWS

## Goal

A minimal guide on how to use OpenTofu to provision an EC2 instance on AWS.

## **Set up AWS**

### **AWS CLI**

In your local machine install AWS CLI, my [guide](https://github.com/francisco-camargo/francisco-camargo/blob/master/src/aws/aws_cli/README.md).

Verify with

```bash
aws --version
```

### **IAM Identity Center**

#### Step 1: Complete Identity Center Setup

1. **Enable IAM Identity Center** in your AWS Console
2. **Choose your identity source** - for a personal account, select "Identity Center directory"
3. **Complete any remaining setup steps** AWS shows you

#### Step 2: Create Your User

1. **In Identity Center, go to "Users"** in the left sidebar
2. **Click "Add user"**
3. **Fill in your details**:

    - Username (your choice)
    - Email address
    - First/Last name
    - Set a password or have AWS generate one

4. **Create the user**

#### Step 3: Create a Permission Set

1. **Go to "Permission sets"** in the left sidebar
2. **Click "Create permission set"**
3. **Select permission set type** select "Custom permission set"
4. **Add inline policy**:

   - In the "Permissions" section, locate "Inline policy"
   - Click "Add inline policy"
   - Switch to JSON editor and paste (_you must remove the comments_):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ec2:*",             // For managing EC2 instances
           "elasticloadbalancing:*",  // For potential load balancing
           "iam:CreateServiceLinkedRole",  // For EC2 service roles
           "iam:PassRole",      // For assigning roles to EC2
           "s3:*"              // For OpenTofu state storage
         ],
         "Resource": "*"
       }
     ]
   }
   ```

   This permission set provides:
   - Full EC2 management capabilities
   - S3 access for OpenTofu state files
   - Minimum IAM permissions for EC2 operation
   - Load balancing capabilities if needed

5. **Configure the permission set**:

    - Name: `EC2-OpenTofu-Access`
    - Description: "Permissions for EC2 management and OpenTofu infrastructure deployment"

6. **Complete the creation** and proceed to Step 4 for assignment

#### Step 4: Assign User to Account

1. **Go to "AWS accounts"** in the left sidebar
2. **Select your AWS account**
3. **Click "Assign users or groups"**
4. **Select your user** and the **permission set** you created
5. **Finish the assignment**

#### Step 5: Get Your SSO Information

In Identity Center, find:

- **AWS access portal URL** (something like `https://d-xxxxxxxxxx.awsapps.com/start`)
- **SSO region** (where Identity Center is enabled)

#### Step 6: Configure AWS CLI

Now run:

```bash
aws configure sso
```

You'll be prompted for:

- **SSO session name**: Pick any name (like "personal" or "main")
- **SSO start URL**: Use the access portal URL from step 5
- **SSO region**: The region where Identity Center is set up
- **SSO registration scopes**: Enter `sso:account:access` (this is the default and minimum required scope)
- **Default client region**: Your preferred AWS region for resources
- **Default output format**: `json` (recommended)

The CLI will open a browser for you to authenticate.

#### Step 7: Test It

```bash
aws sso login --profile <sso profile>
```

Even if I granted STS permissions in the json above, I was not able to get the following to work even after successful SSO CLI login

```bash
aws sts get-caller-identity --profile <sso profile>
```

- Create or use existing Access Key ID and Secret Access Key

### Step 8: SSH Credentials

Create an SSH key pair that will allow you to securely connect to the EC2 instance we'll create with OpenTofu.

#### **Generate SSH Key Pair**

Navigate to the `tofu` directory and create the key pair there:

```bash
cd tofu
aws ec2 create-key-pair --key-name ec2-key --query 'KeyMaterial' --output text --profile <sso profile> > ec2-key.pem
```

**Note:** Replace `<sso profile>` with the actual profile name you configured during the SSO setup.

**Important:** The key pair name in AWS will be `ec2-key` (without .pem), but the local file is saved as `ec2-key.pem`. In your `terraform.tfvars` file, use just `ec2-key` for the `key_name` variable, not `ec2-key.pem`.

#### **Set Key Permissions** (Windows)

This secures the private key file with proper permissions - only your user account can read it. This is required for SSH clients to accept the key.

**In Git Bash/MINGW64:**

```bash
icacls ec2-key.pem /inheritance:r
icacls ec2-key.pem /grant:r "$USERNAME":"(R)"
```

**In PowerShell:**

```powershell
icacls ec2-key.pem /inheritance:r
icacls ec2-key.pem /grant:r "%USERNAME%":"(R)"
```

**Alternative method (works in any terminal):**

```bash
chmod 400 ec2-key.pem
```

**⚠️ CRITICAL SECURITY WARNING**: Never share or expose your private SSH key contents. If a private key is compromised (shared in chat, committed to public repositories, etc.), it must be immediately deleted and replaced. Private keys provide direct access to your servers and should be treated as highly sensitive credentials.

## OpenTofu Infrastructure-as-Code

OpenTofu roadmap to get an EC2 instance running:

### Phase 1: Local Setup

1. **Install OpenTofu** on your Windows machine
    [Guide](https://opentofu.org/docs/intro/install/windows/). Not sure if adding it to PATH helped or not

    ```powershell
    winget install --exact --id=OpenTofu.Tofu
    ```

    Restart the terminal, then verify the installation:

    ```bash
    tofu -version
    ```

2. **Create Project Structure**

    Create a subdirectory called `tofu` to keep your configuration files organized:

    ```bash
    mkdir tofu
    cd tofu
    ```

    Inside this directory, create these files:
    - `provider.tf` - AWS provider configuration
    - `main.tf` - EC2 instance definition
    - `variables.tf` - Customizable parameters
    - `outputs.tf` - Output values like IP address
    - `terraform.tfvars` - Your specific variable values

### Phase 2: Write Configuration Files

1. **Provider Configuration (provider.tf)**

    This file configures the AWS provider for OpenTofu, specifying which AWS region to deploy resources in and how to authenticate. It sets up the connection between OpenTofu and your AWS account.

2. **Main Resources (main.tf)**

    The core infrastructure file that defines your AWS resources. For this project, it includes the EC2 instance configuration and a security group that allows SSH access. This is where you define the actual cloud resources you want to create.

3. **Variables (variables.tf)**

    Defines input variables that make your configuration flexible and reusable. This file declares variables like region, AMI ID, instance type, and SSH key name, allowing you to customize your deployment without changing the core resource definitions. **This file is committed to the repository** and contains safe default values.

4. **Outputs (outputs.tf)**

    Specifies values to be displayed after OpenTofu applies your configuration. This file will define outputs like the instance's public IP address and a ready-to-use SSH command, making it easier to connect to your newly created instance.

5. **Custom Values (terraform.tfvars)**

    Contains the specific values for variables defined in variables.tf. This is where you set your preferred region, instance size, SSH key name, and most importantly, your actual public IP address for secure SSH access. **This file is NOT committed to the repository** (excluded by .gitignore) because it contains personalized and potentially sensitive information like your public IP address. You should customize the values in this file to match your specific requirements and security needs.

    **Key naming:** Make sure the `key_name` in `terraform.tfvars` matches the key pair name you created in AWS (e.g., `ec2-key`), not the filename (e.g., `ec2-key.pem`).

**Important**: The `terraform.tfvars` file should contain different values from the defaults in `variables.tf`. For example:

- `variables.tf` might have a generic AMI ID and open SSH access (`0.0.0.0/0`)
- `terraform.tfvars` should have the correct AMI for your chosen region and your actual public IP (`x.x.x.x/32`) for secure SSH access

### Security: .gitignore Configuration

This repository includes a `.gitignore` file that prevents sensitive files from being committed to version control. This is crucial for security when working with infrastructure-as-code tools like OpenTofu.

**Files excluded from the repository:**

- `*.tfvars` - Contains your personal configuration values including IP addresses
- `*.tfstate*` - State files that contain all resource details and can include sensitive data
- `*.pem`, `*.key` - SSH private keys
- `.terraform/` - Provider binaries and cached modules
- `*.tfplan` - Plan files that may contain sensitive output

**Why this matters:**

- **Security**: Prevents accidental exposure of your public IP, SSH keys, or AWS resource details
- **Privacy**: Keeps your personal configuration separate from the shared codebase
- **Best Practice**: Follows standard Terraform/OpenTofu security guidelines

**What gets committed:**

- `variables.tf` - Variable definitions with safe defaults
- `main.tf`, `provider.tf`, `outputs.tf` - Infrastructure code
- `README.md` - Documentation
- `.terraform.lock.hcl` - Dependency lock file (contains version info, no sensitive data)

This approach allows you to safely share infrastructure code while keeping sensitive configuration local to your machine.

**Note about .terraform.lock.hcl:** This file should be committed as it ensures all team members use the same provider versions, improving consistency and reproducibility. It contains only version numbers and checksums, no sensitive information.

### Phase 3: Deployment

1. **Configure AWS Authentication**

    Before running OpenTofu commands, you need to authenticate with AWS. Since you're using SSO, run:

    ```bash
    aws sso login --profile <sso profile>
    ```

2. **Initialize OpenTofu**

    Navigate to the `tofu` directory and initialize:

    ```bash
    cd tofu
    tofu init
    ```

    This downloads required providers and modules.

3. **Plan Deployment**

    Preview what will be created:

    ```bash
    tofu plan
    ```

    **Optional: Save the plan to a file**

    You can save the execution plan to a file for later use:

    ```bash
    tofu plan -out=tfplan
    ```

    This creates a binary file called `tfplan` that contains the exact execution plan. Benefits:
    - **Review before apply**: You can examine the plan file and apply it later
    - **Consistency**: Ensures the same plan is applied even if configuration changes
    - **CI/CD workflows**: Useful in automated deployment pipelines
    - **Safety**: Prevents surprises between planning and applying

    To apply a saved plan:

    ```bash
    tofu apply tfplan
    ```

    **Note:** Plan files are excluded from git (via .gitignore) as they can contain sensitive information.

4. **Apply Configuration**

    Create the resources:

    ```bash
    tofu apply
    ```

    Or apply a saved plan:

    ```bash
    tofu apply tfplan
    ```

    Type `yes` when prompted to confirm (not needed when applying a saved plan).

5. **Connect to Your Instance**

    After successful deployment, use the SSH command from the outputs:

    ```bash
    ssh -i ec2-key.pem ec2-user@<public_ip>
    ```

    **Finding connection information:**

    The public IP and other instance details are automatically stored in the `terraform.tfstate` file, which is created in the `tofu` directory after running `tofu apply`. This state file contains the current state of all your infrastructure resources. You can also view the outputs by running:

    ```bash
    tofu output
    ```

    This will display the formatted outputs including the ready-to-use SSH command.

    **Troubleshooting SSH connection issues:**

    If you get a "Connection timed out" error when trying to SSH:

    1. **Check for SSH key format issues** - If you get "load pubkey: invalid format", this is often just a warning and not a failure. The connection may still work:

       ```bash
       # This warning is usually harmless, if you see the host authenticity prompt,
       # it means SSH is connecting successfully. Type 'yes' to continue.

       # If you want to suppress the warning:
       ssh -i ec2-key.pem -o IdentitiesOnly=yes ec2-user@<public_ip>

       # Or check if the key file is corrupted:
       file ec2-key.pem  # Should show "PEM RSA private key"
       ```

       **Host authenticity prompt**: When you see "The authenticity of host... can't be established", type `yes` to accept and continue. This is normal for first-time connections.

    **Disconnecting from SSH:**

    Once you're connected to your EC2 instance, you can disconnect using:

    ```bash
    # Method 1: Type exit command
    exit

    # Method 2: Use keyboard shortcut
    # Press Ctrl+D

    # Method 3: If session is unresponsive
    # Press Enter, then type ~. (tilde followed by period)
    ```

    1. **Check file locations** - Both the SSH key file and OpenTofu files are now in the same directory:
       - SSH key: `tofu/ec2-key.pem`
       - OpenTofu files: `tofu/` directory

       Make sure you're running the SSH command from the `tofu` directory where both the `.pem` file and state files are located:

       ```bash
       # From the tofu/ directory:
       ssh -i ec2-key.pem ec2-user@<public_ip>
       ```

    2. **Check your security group configuration** - Ensure your `terraform.tfvars` has the correct `allowed_ssh_cidr` value:

       ```hcl
       allowed_ssh_cidr = "YOUR_PUBLIC_IP/32"
       ```

       Find your public IP at [whatismyipaddress.com](https://whatismyipaddress.com)

    3. **Verify the instance is running** - Check the AWS console or run:

       ```bash
       aws ec2 describe-instances --instance-ids <instance-id> --profile <sso profile>
       ```

    4. **Check file permissions** - Ensure the SSH key has correct permissions:

       ```bash
       ls -la *.pem
       # Should show: -r-------- (chmod 400)

       # Also verify the file format:
       file ec2-key.pem
       # Should show: "PEM RSA private key" or similar
       ```

    5. **Verify key name matches** - Check that the key name in `terraform.tfvars` matches what was created:

       ```bash
       aws ec2 describe-key-pairs --profile <sso profile>
       ```

    6. **Wait for instance initialization** - New instances can take a few minutes to fully boot and accept connections

    7. **Update security group if needed** - If you need to change the allowed IP, update `terraform.tfvars` and run:

       ```bash
       tofu plan
       tofu apply
       ```

### Phase 4: Cleanup

When you're done experimenting, destroy the resources to avoid unnecessary charges:

```bash
tofu destroy
```

Type `yes` when prompted to confirm.

#### **Optional: Complete cleanup (if you want to start fresh)**

If you need to delete SSH keys due to security concerns or want to completely clean up:

```bash
# From the tofu/ directory:
# Delete the key pair from AWS
aws ec2 delete-key-pair --key-name ec2-key --profile <sso profile>

# Delete local SSH key files
rm ec2-key.pem
```

**Note:** Only run these commands if you're certain you no longer need the SSH keys. If you accidentally exposed your private key (e.g., committed it to version control), you should immediately delete and recreate your keys for security.

## Next Steps

Once you're comfortable with this basic setup, you can explore:

- Adding more EC2 configuration options
- Setting up autoscaling groups
- Creating a proper networking setup with VPC and subnets
- Implementing remote state storage in S3
- Adding state locking with DynamoDB
- Implementing remote state storage in S3
- Adding state locking with DynamoDB
- Adding state locking with DynamoDB
