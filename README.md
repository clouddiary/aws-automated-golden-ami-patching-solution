
![image](https://user-images.githubusercontent.com/114372599/205932239-014f90d5-af28-4a7d-b695-375704eabff5.png)

For better understanding & to keep it short, I am spliting whole solution into two parts.

#### Part-1 : Will explain environment setup & define problem statement
#### Part-2 : Will discuss about end to end solution & challenges

Operating System Patching has always been a management overhead for many organization. It becomes more trivial when organizations uses hardened/golden AMI to create machines for various compliance & regulatory (PCI DSS/PHI, etc.) requirements. In this blog I will use a fictious company XYZ Corporation to highlight those OS patching challenges & possible solutions.

#### XYZ Corporation environment setup

XYZ Corporation(Fictional) a multinational drug manufacturing company have multiple AWS Accounts. For regulatory & compliance reasons, it created multiple hardened/Golden AMI's (Linux/Windows, etc.) in one account (Let's assume QA account) & shared it with other accounts (UAT/PROD) for creating EC2 instances or Autoscaling Groups.

![env](https://user-images.githubusercontent.com/114372599/205928412-5d9ab25e-a1e5-457f-89ef-327b84eff87e.png)


	1. System Admin hardened multiple AMIs based on various security requirements & shared those hardened AMI with UAT & PROD accounts.
	2. System Admin copied hardened AMI Id in parameter store in all accounts with a specific keys to identify each hardened/golden ami uniquely.
	3. All Infrastructure DevOps pipeline/CloudFormation script used this centralized param store keys to get specific hardened AMI Ids.
	4. Within few months XYZ Corporation AWS Accounts had 1000's of stack (ASGs/EC2) in each account to support various business demands.
	5. With such a large volume, Operations team felt need for some automation process to streamline monthly patching. 
	6. Operations team decided to use AWS native patching service called SSM Patch Manager for monthly patching.
	7. They decided to patch QA account (~1000 EC2 instances) on 1st Saturday & UAT account (~1500 EC2 instances) on 2nd Saturday & PROD account (~2000 EC2 instances) on 3rd Saturday of every month.
	8. Maintenance window (SSM feature that lets schedule patching process) & patch base lines (SSM feature that lets us configure what all types of patches to install) were created & configured accordingly.
	9. Operations team followed AWS documentation (link) & started using AWS SSM Patch Manager for monthly patching.
	
AWS SSM Patch Manger (Short Overview): AWS SSM Patch Manager automates the process of patching Windows and Linux managed instances. AWS SSM agent(needs to be installed in each instance) scans your instances for missing patches or scan and install missing patches. Patch Manager uses patch baselines, which include rules for auto-approving patches within days of their release, as well as a list of approved and rejected patches. You can install patches on a regular basis by scheduling patching to run as a Systems Manager maintenance window task. You can install patches individually or to large groups of instances by using Amazon EC2 tags. 

![ssm](https://user-images.githubusercontent.com/114372599/205929423-6724bfd3-839a-4cd4-80fc-dc98dc2d3125.png)

AWS Patch Manager helped XYZ Corporations to automate monthly patching process where running instances were patched once in a month. Security & Operations team was using Patch Manager Dashboard to monitoring patch compliance stats every month. Even after smooth monthly patching, They were struggling to get correct patch compliance status(due to inconsistency). After few months they discovered listed inconsistency issues & root cause behind them.

#### Issues
	1. Many monthly patched instance which were showing as complaint after monthly patching suddenly disappears In SSM Patch Manager.
	Root cause : Upon investigation Operations team discovered many Auto Scaling Groups(ASG) recycled boxes(for various reasons) which resulted in termination of patched EC2 instances & since new instances came with golden AMI(hardened AMI), its missing those new patches.
	2. Many monthly patched instances were showing up in SSM Patch Manager but not as complaint.
	Root cause : Monthly patching itself failed on such instances since 
			i. SSM agent(which scans & installs missing patches) is either missing or in stopped state. 
			ii. SSM Agent lost connectivity to AWS SSM patch manager service.
			iii. EC2 instance didn't had enough capacity (CPU/Disk) to perform patching operations.
	
Operations team tried various approaches to fix identified issues but none of them helped & finally they concluded XYZ Corporation needs a process to build patched golden AMI once in a month so that even if ASG launches new instances they all will be compliant on SSM Patch Manager. Here is steps taken

#### Steps
	1. Initially operations team decided to patch & create new golden AMI manually once in a month.
	2. Shared new golden AMIs with all other accounts.
	3. Updated each SSM Param Store keys with new golden AMI Id.
 But Wait!! How above 3 step would fix core issue (which is letting ASGs know new golden AMI Id to launch any new instances)? 
 Updating param store key with new golden AMI Id will not help in propagating new golden AMI Id to existing ASGs rather it will help only for new ASG create/update scenario. The core issue still not resolved & XYZ Corporation had to find a way to propagate all new golden AMI across all accounts. 

#### Here are some challenges identified in propogating new golden AMI :
	1. XYZ Corporations had 1000s of ASG in each account using golden AMIs so it was massive work to propagate new golden AMI across all accounts. Especially production account where applications are serving live traffic.
	2. All these ASG in each account were created via DevOps pipelines/CloudFormation so it was important to make sure golden AMI propagation work shouldn’t break any CI/CD process.

I will conclude part-1 here for you to understand XYZ corporation environment & think of possible solutions. Will be back soon with part-2 discussing possible solutions & challenges. 
