## Setup

Log in to the [Unified Demo Framework Portal](https://udf.f5.com)

Deploy the DevOps Base blueprint

> For more information on deploying a UDF blueprint, please reference the [UDF documentation](https://help.udf.f5.com/en/)

Under components, click the access dropdown on the client system, then click VS CODE

Open a new [terminal](https://code.visualstudio.com/docs/editor/integrated-terminal) in VS Code

## Set the BIG-IP Password

Set the BIG-IP password as an environment variable:

>  Obtain the BIG-IP password on the BIG-IP1 and BIG-IP2 documentation pages inside the UDF deployment
        
```bash
export bigip_pwd=replaceme
```

## Fork the lab repository

The labs for this UDF blueprint can be found on [GitHub](https://github.com/f5devcentral/automate-one-thing).  You will need to fork this repository into your own GitHub environmnent. 

For information on how to do this, please visit the [GitHub documentation page](https://help.github.com/en/github/getting-started-with-github/fork-a-repo#fork-an-example-repository).

## Checkout Forked Repository

To conduct the labs, you will need to check our your forked version of the repository:

```bash
cd ~/projects

# replace your GitHub username
git clone https://github.com/<githubusername>/UDF-DevOps-Base.git
```

## Change into the Lab0 directory:

```bash
cd ~/projects/UDF-DevOps-Base/labs/lab0
```

## Get Latest Version of the Lab

The labs in the UDF-DevOps-Base repository are constantly being updated and new labs are being added.  If it has been awhile since you forked the repository then now is a good time to update your local copy:

```bash
git remote add upstream https://github.com/f5devcentral/UDF-DevOps-Base.git
git fetch upstream
git merge upstream/main
```

## Install BIG-IP ATC and FAST extension
In preperation for further labs we need to install the F5 Automation Toolchain and the F5 Application Service Templates extensions.

### Set the ATC versions as an environment variables:

>  Obtain the BIG-IP password on the BIG-IP1 and BIG-IP2 documentation pages inside the UDF deployment

```bash
export as3_version=3.19.1
export do_version=1.12.0
export ts_version=1.11.0
export fast_version=1.0.0
```

### Install ATC on BIG-IP1 and BIG-IP2:

The client machine has the F5-CLI docker container installed.  We will use this container to install and interact with the 

>  If the F5-CLI Docker container is not running, start it with the 'docker start f5-cli' command

```bash
for i in {6..7} 
do
    # authenticate to the BIG-IP
    docker exec -it f5-cli f5 login --authentication-provider bigip --host 10.1.1.$i --user admin --password $bigip_pwd

    # install the do, as3 and ts extensions
    docker exec -it f5-cli f5 bigip extension do install --version $do_version
    docker exec -it f5-cli f5 bigip extension as3 install --version $as3_version
    docker exec -it f5-cli f5 bigip extension ts install --version $ts_version
done
```

### Install FAST on BIG-IP1 and BIG-IP2:

```bash
for i in {6..7} 
do
    # move to a temporary directory 
    cd /tmp

    # set the FAST RPM name
    FN=f5-appsvcs-templates-1.0.0-1.noarch.rpm
    
    # download the FAST RPM
    wget https://github.com/F5Networks/f5-appsvcs-templates/releases/download/v$fast_version/$FN
    
    # set information for curl
    CREDS=admin:$bigip_pwd
    IP=10.1.1.$i
    LEN=$(wc -c $FN | awk 'NR==1{print $1}')

    # upload the FAST RPM
    curl -kvu $CREDS https://$IP/mgmt/shared/file-transfer/uploads/$FN \
    -H 'Content-Type: application/octet-stream' \
    -H "Content-Range: 0-$((LEN - 1))/$LEN" -H "Content-Length: $LEN" \
    -H 'Connection: keep-alive' --data-binary @$FN

    # install the FAST RPM
    DATA="{\"operation\":\"INSTALL\",\"packageFilePath\":\"/var/config/rest/downloads/$FN\"}"
    curl -kvu $CREDS "https://$IP/mgmt/shared/iapp/package-management-tasks" \
    -H "Origin: https://$IP" -H 'Content-Type: application/json;charset=UTF-8' --data $DATA
done
```

## Test the ATC and FAST Extensions
An important, but often overlooked, part of automation is the creation of test cases to ensure the automation achieved the desired outcome. In this lab, we will leverage the Chef [InSpec][InSpec] tool.  InSpec is based on Ruby and provides a framework to test infrastructure deployed in a public or private cloud.

>  You should see the InSpec test run twice, once for each BIG-IP, and all tests should be green.

```bash
cd ~/projects/UDF-DevOps-Base/labs/lab0
for i in {6..7} 
do
    inspec exec https://github.com/f5solutionsengineering/big-ip-atc-ready.git \
    --input bigip_address=10.1.1.$i password=$bigip_pwd do_version=$do_version \
    as3_version=$as3_version ts_version=$ts_version fast_version=$fast_version
done
```

## Onboard the BIG-IP

Now that the BIG-IP1 and BIG-IP2 ATC and FAST extension tests have passed, it's time to perform the onboarding steps.  To onboard the BIG-IP appliances we will leverage the F5 Automation Toolchain's [Declarative Onboarding][DO] extension.

### Examine the DO declaration

Open the _bigip1.do.json_ declaration and examine the onboarding settings.  Some settings to note:

- how the hostname is set
- how the name servers are set
- how BIG-IP modules are provisioned
- how VLANS and Self-IPs are configured

### Add a Default User Shell

While the provided DO declaration is a good starting point, it is missing one important configuration that allows us to run InSpec tests against the BIG-IP.  The admin user needs to have a bash shell instead of the default TMSH shell. 

#### Set admin user's default shell
For this step, I want you to use the [Declarative Onboarding][DO] documentation located on [clouddocs.f5.com](https://clouddocs.f5.com) to set the admin user's default shell to bash.

> The DO schema object you need to add is _User_

#### Testing your answer
When you're ready to test your solution run the following command:

>  the test should output true for each declaration

```bash
bash check_my_answer.sh
``` 

If you are stuck on this exercise, there is an example<sup>[1](#doexample)</sup> in the [Declarative Onboarding][DO] documentation.

Once you have updated the admin user's default shell in both _bigip1.do.json_ and _bigip2.do.json_ files and the test script passes you are ready to proceed to the next step. 

### Onboard BIG-IP1 and BIG-IP1:

```bash
COUNTER=1
for i in {6..7}
do  
    # authenticate to the BIG-IP
    docker exec -it f5-cli f5 login --authentication-provider bigip \
    --host 10.1.1.$i --user admin --password $bigip_pwd

    # POST the DO declaration
    docker exec -it f5-cli f5 bigip extension do create --declaration \
    /f5-cli/projects/UDF-DevOps-Base/labs/lab0/bigip$COUNTER.do.json

    # increment counter
    let COUNTER=COUNTER+1
done
```

### Test the Onboarding Process

Run the following InSpec profile to test that the BIG-IP is correctly configured:

>  You should see the InSpec test run twice, once for each BIG-IP, and all tests should be green.

```bash
for i in {6..7}
do
    inspec exec do-ready -t ssh://admin@10.1.1.$i --password=$bigip_pwd \
    --input internal_sip=$10.1.10.$i external_sip=10.1.20.$i 
done
```

[DO]: https://clouddocs.f5.com/products/extensions/f5-declarative-onboarding/latest/

## Conclusion
You have successfully completed Lab0 and your environment is now ready to run other labs.  Have fun and good luck!

---

#### Footnotes:
<small><a name="doexample">1</a>: Declarative Onboarding example of setting the admin users shell can be found [here](https://clouddocs.f5.com/products/extensions/f5-declarative-onboarding/latest/bigip-examples.html?highlight=bash#standalone-declaration).  Note, you do not need the password attribute from this example. <small>