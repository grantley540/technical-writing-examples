# BindRetrofit

#### Automated VPC Resolver retrofits for classic and guardrails accounts

## What
This script will retrofit AWS Route53 VPC Resolvers into a classic or guardrails VPC.  This means that DNS services provided by the VPC Binds are replaced with the native AWS VPC resolver functionality.  This essentially turns DNS in VPCs into a serverless service. DNS services provided by BINDs has proven to be very unreliable and troublesome so replacing them with a native, serverless AWS service is a massive improvement.

The best case scenario when running this script is a DNS outage in the VPC for a few minutes.  The worst case scenario is a DNS outage that lasts until a person manually either rolls back the retrofit or fixes forward.  Instructions for rolling back are provided in the Rollback section.

You should work with the VPC owners before running this script in a VPC to ensure the owners are prepared for a brief DNS outage (with the usual caveat that the outage could last longer if issues are encountered).

High level process of the script:

1. Collect information from the VPC and perform some basic sanity checks
2. Retrieve the VPC Bind configuration (forwarding rules) from the someurl.com table
3. Build the cloudformation template for the VPC resolver infrastructure including the forwarding rules retrieved in step 2.
4. Terminate the Binds in the VPC and delete the ENIs
5. Create the cloudformation stack which sets up the VPC resolver infrastructure
6. Create the test DNS cloudformation stack to verify there is DNS in the VPC
7. Delete the test DNS cloudformation stack

The log output of the script and the generated cloudformation (step 3) are stored in the `./retrofit/<Account Alias>` folder.  This data needs to be committed back to this repo after running the script in order to preserve the cloudformation for future updates.

## How
### Requirements
1. Docker
2. An AWS profile in your `~/.aws/credentials` file for <> account for the us-east-1 region.  This profile should have <> or equivalent permissions. Use <> or similar tool to generate these profile creds
3. An AWS profile in your `~/.aws/credentials` file for the target retrofit account for the region you will be retrofitting.  This profile should have <> or equivalent permissions. Use <> or similar tool to generate these profile creds

### Steps
**WARNING**:  
You should be prepared to relaunch BIND services in the retrofit account (using terraform) before you begin a retrofit in the unlikely event that the new VPC resolvers do not work and can't be readily fixed.  A relaunch of the BIND servers must preserve the previous BIND server IPs.  If the BINDs are relaunched with new IP addresses, DNS service in the VPC will not be fully restored and applications would need to be restarted to start working again.  If you aren't sure if the terraform has been updated to reuse the previous IPs, do not proceed and find the nearest adult.  

**NOTE**:  
As a prerequisite, you should check the dynamodb `bind-config` table in the <> account and see if the account you want to retrofit has an entry in the table.  If it does, make sure that the rules are as expected because bindretrofit uses whatever is in `bind-config` on the new vpc resolvers.  If there is no entry in `bind-config`, bindretrofit will use the default rules in the `000000000000 vpc-00000000 default-new` entry.  If there _is_ an entry for the account but you want to use the default rules you can just delete the retrofit account entry.  

1. Clone this repo
2. From within the repo, run `make run`
   * This builds a docker container and drops you in it
   * Your local `~/.aws/credentials` file is mounted in `/root/.aws/credentials`
   * This repo itself is mounted in `/app/bindretrofit/`

   **NOTE**:<br> If you get a docker error similar to `filesystem layer verification failed for digest sha256:28d6cdc18712a7c9061a7c3xxxxxxxxxxxx` you will
   have to move over to `Internet` wireless and remove your docker proxy. (on mac right click the docker icon and go to 'Preferences' -> 'Proxies').  Then run `make run` again and
   you should get past that error.

   This error is caused by the proxy.  Once you pull the image the first time you shouldn't need to move to `Internet` wireless again unless the image gets deleted from your laptop.
3. Generate creds (using <> or similar tool) for the 2 profiles mentioned in 'Requirements' above
4. Perform a bindretrofit retrofit by running the bindretrofit python script. (you should be inside the docker container)
	```
	Usage:
	  bindretrofit.py [--debug] [--dryrun] [--rerun] XXPROFILE TARGET_ACCOUNT_PROFILE TARGET_ACCOUNT_NUMBER TARGET_ACCOUNT_REGION
	```

   For example:  
   ```
   python bindretrofit.py XXX XXX 000000000000 us-east-1
   ```

   **NOTE**:<br> If no eni is found associated with the private ip, make sure the eni's associated with the binds are tagged with XX

5. Check the output of the script.  If any errors have occured after changes were made, see the Rerun and/or Rollback sections below.
6. Stop/Start the NAT instance in the VPC to ensure it starts using AWS internal DNS resolvers.  (we've seen some issues with squid using the old BIND IPs)
7. Exit the container and commit the changes made in the repo in `./retrofit/` to the `master` branch.  
   **NOTE**:<br> Committing the changes in the repo is crucial in order to capture the generated cloudformation and logs in source control.

## Rerun
In the event that an error is encountered after the BINDs have been terminated, it is possible to rerun bindretrofit to try and resolve the error. If the error occurs during the `GR-Route53-VPCResolver-Stack` stack creation itself and the stack fails to successfully create, this stack needs to be deleted first, before doing a rerun.  If the error occurs during testing and the `GR-Route53-VPCResolver-Stack` stack was successfuly created, DO NOT do a rerun as it's not going to fix anything.

Doing a rerun simply reuses the previously collected information and previously generated cloudformation to relaunch the cloudformation stack.  It simply skips all the initial data gathering and template creation steps.  The alternative to a rerun is to relaunch the BINDs using terraform by following the "Rollback" section below and then rerunning bindretrofit without the `--rerun` option just as you would during the very first bindretrofit retrofit attempt.

### Rerun Steps

1. Ensure the `GR-Route53-VPCResolver-Stack` cloudformation stack in the target account/region is completely deleted (do this from the console)
2. Generate creds (using XX or similar tool) for the 2 profiles mentioned in 'Requirements' above
3. Rerun bindretrofit using the `--rerun` option.
   For example:  
   ```
   python bindretrofit.py --rerun XX XX 000000000000 us-east-1
   ```
4. Stop/Start the NAT instance in the VPC to ensure it starts using AWS internal DNS resolvers.  (we've seen some issues with squid using the old BIND IPs)
5. Exit the container and commit the changes made in the repo in `./retrofit/` to the `master` branch.  
   **NOTE**:<br> Committing the changes in the repo is crucial in order to capture the generated cloudformation and logs in source control.

## Rollback
In the event that you want to rollback the resolver retrofit and relaunch BIND servers, follow these instructions

The specific changes that this script makes which may need to be rolled back to restore BIND DNS services include:
1. Both the BIND servers terminated
2. Both the BIND ENI's deleted
3. The VPC updated to use a new DHCP options set

The classic/guardrails accounts Terraform (and XX (guardrails accounts), XX (classic accounts)) is required to be ran in order to restore the BIND servers. How to build BIND service terraform in an account is outside of the scope of this document but you **MUST** ensure that the new BINDs get the same IPs that they had prior to the retrofit.

### Rollback Steps
These steps assume a failure in the cloudformation deployment, which means all 3 changes mentioned above took place and need to be rolled back.  If a failure occurs before the cloudformation deployment, adjust the required rollback steps accordingly.

1. Delete the cloudformation stack `GR-Route53-VPCResolver-Stack` (do this from the console)  
2. Set the VPC DHCP options back to the original DHCP options set (do this from the console).  The original DHCP options set ID is provided in the script output
3. `Terraform destroy` the BIND terraform in the account/region
4. `Terraform apply` the BIND terraform in the account/region.  **NOTE**: the terraform should be updated to support relaunching the BINDs with the previous IPs.

## Troubleshooting
* If DNS doesn't work in the account after successfully running bindretrofit, here are some possible things to check:  
  * The XXX DNS forwarding servers used in the resolver rules may not have access through the XXX firewall.  The list of possible forwarder servers in use in accounts is:  

    ```
    X.X.X.X  
    X.X.X.X   
    ```
    OR    
    ```
    X.X.X.X   
    X.X.X.X   
    ```

    You can try changing the forwarders in the someurl.com and XXX.com resolver rules to the set not currently in use. (via the console -> route53).  For example, if X.X.X.X  and X.X.X.X are the forwarders for someurl.com rule you can try switching them to X.X.X.X  and X.X.X.X  and deleting the old ones or vice versa.  If this fixes the issue, be sure to update the generated cloudformation to reflect this change as it would get reverted back if the cloudformation stack was re-run in the future.

* If the resolver stack fails to create with an 'internal error' on the resolver endpoint and mention of 'failure to stabilize' try waiting about 15-20 minutes and then doing a "rerun" of bindretrofit.  See the "Rerun" section above for instructions.  This error seems to happen most frequently when back to back retrofits/rollbacks occur (such as during testing while developing bindretrofit).

## Making Changes
After running bindretrofit in an account, you may (in the future) need to modify resolver rules or other things associated with route53 resolvers.  Changes can be made by modifying the `generated_cloudformation.yaml` file in `retrofit/<your account here>`.  The deployed route53 resolver stack can then be updated by deploying the updated template file.  The command to update the stack would look like this:

```
aws --no-verify --region YOUR_REGION_HERE --profile YOUR_PROFILE_HERE cloudformation update-stack --stack-name GR-Route53-VPCResolver-Stack --template-body file://retrofit/YOUR_ACCOUNT_HERE/generated_cloudformation.yaml --parameters file://retrofit/YOUR_ACCOUNT_HERE/cloudformation_params.json --tags file://retrofit/YOUR_ACCOUNT_HERE/cloudformation_tags.json
```
If you need to update the tags or the stack params of the resolver cloudformation stack, you can update the `retrofit/YOUR_ACCOUNT_HERE/cloudformation_params.json` and `retrofit/YOUR_ACCOUNT_HERE/cloudformation_tags.json` files.  You will need to reference these files in your cloudformation command.

## Important design considerations
The BINDs have to be terminated and their ENIs deleted in order to free up their private IPs.  These IPs are used for the inbound vpc resolver forwarders.  This is necessary to enable the existing BIND IPs that all applications in a VPC are using to continue to resolve DNS queries.  If the IPs weren't reused, all applications in a VPC would need to be restarted in order to pick up the new VPC DNS configuration.

The bind-config table is used as the source of truth for all DNS forwarding rules in a VPC.  If a VPC isn't present in this table, the default ruleset from the table is used.  It is possible that there are custom forwarding rules in a VPC that weren't captured in the bind-config table and thus won't be present in the new VPC resolvers and thus certain DNS queries may not work or behave as expected after the retrofit.  

The cloudformation that this script generates and runs should be stored in source control so as to allow updates to be made in the future.  If new forwarding rules need to be added, for example, they should be added to the cloudformation template and the stack should be updated.

This script is only meant to be run one time in an account and then never again (so long as there were no errors/failures).

The DNS name someurl.com was created to test internal XXX DNS resolution

## Terraform updates
Instructions for making terraform updates to support static IPs on the BINDs for rollbacks
#### GR Accounts
  * Update `XX/<gr account alias>/services/bind/bind.tf` file  

    At the bottom of the `module "grbind_builder" {` block, add the following additional variables:
    ```
    bind1_ip           = "${var.bind1_ip}"
    bind2_ip           = "${var.bind2_ip}"
    ```

  * Update `XX/<gr account alias>/base.tf` file  

    At the bottom, add the following additional variables (substituting the BIND IPs for your specific account):
    ```
    variable "bind1_ip" {
      type    = "string"
      default = "IP ADDRESS HERE"
    }

    variable "bind2_ip" {
      type    = "string"
      default = "IP ADDRESS HERE"
    }
    ```
#### Classic Accounts
Most classic accounts have already had their BINDs retrofitted and thus the necessary updates were already made to use static IPs on the BINDs.  If there are `Bind-Primary-Retrofit` and `Bind-Secondary-Retrofit` folders in `AWS/<classic account alias>/ManagementHosts` then everything should be setup correctly.  If the folders don't exist then they will need to be created.  Refer to the folders and files in another account that does have them to create them.

#### Ancient Accounts
Here's some notes for doing a retrofit on an ancient ind account that had only one Bind and no formally named mgmt subnets:

* Copy a previous retrofit folder in `/retrofit` and rename it
* Update ALL the params in the `cloudformation_params.json` file and make sure you are reusing the existing Bind IP(s)
* Update the rules as needed (including rule associations) and the multiple references to a hardcoded vpc id in `generated_cloudformation.yaml`
* Make sure there is a `collected_info.yaml` file in the folder.  All that needs to be in there are updated `account_alias` and `region` fields
* If there's no formally named management subnets and there's only one Bind, check the subnet that the bind is in and use that for one of them.  For the second subnet, you can look at the subnets using the last few cidr ranges of the VPC cidr block since those were traditionally the mgmt subnets. Make sure they have the same route tables/NACLs such as `rt-Private` and `nacl-Private`.  Pick an available IP from the second subnet to use. If there's two binds, you should just be able to use those two subnets and existing IPs
* Make sure the NACLs in the chosen mgmt subnets permit tcp and udp port 53 inbound and outbound to X.X.X.X/8 and/or X.X.X.X/8
* Check what forwarders are used on the existing Bind(s) by either looking for an entry in the `bind-config` table or logging into the Bind if there's no entry.  If the config only uses a XXX forwarder, the XXX forwarders may not be opened through the firewall.  In this case the `generated_cloudformation.yaml` rules would need to be updated prior to running bindretrofit or the rules could be updated from the console, afterwards.
* Terminate the existing Bind(s) to free up the IP(s)
* Run bindretrofit with the `--rerun` flag OR just manually create the cloudformation stack using the `tags`, `params` and `generated_cloudformation.yaml` files
* The test stack may fail for various reasons.  Always spin up a XX and test things manually.  Make sure you can resolve internal XXX domains such as someurl.com as well as public domains, from BOTH resolver endpoints.
