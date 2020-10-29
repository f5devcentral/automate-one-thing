# Quickstart 
Use the steps below to bypass Lab 0. 

## Access the lab environment
Log in to the [Unified Demo Framework Portal](https://udf.f5.com)
Deploy the DevOps Base blueprint

> For more information on deploying a UDF blueprint, please reference the [UDF documentation](https://help.udf.f5.com/en/)

Under components, click the access dropdown on the client system, then click VS CODE

Open a new [terminal](https://code.visualstudio.com/docs/editor/integrated-terminal) in VS Code

## Set the BIG-IP Password

Set the BIG-IP password as an environment variable:

> Obtain the BIG-IP password on the BIG-IP1 and BIG-IP2 documentation pages inside the UDF deployment
    
```bash
export bigip_pwd=replaceme
```

## Fork the lab repository
The labs for this UDF blueprint can be found on [GitHub](https://github.com/f5devcentral/automate-one-thing).  You will need to fork this repository into your own GitHub environmnent. 

For information on how to do this, please visit the [GitHub documentation page](https://help.github.com/en/github/getting-started-with-github/fork-a-repo#fork-an-example-repository).


## Check out the forked repository 
To conduct the labs, you will need to check our your forked version of the repository:

### SSH Access
SSH into the client instance (you can do this via the Access drop down in the UDF portal)

```bash
cd ~/projects

# replace your GitHub username
git clone https://github.com/<githubusername>/UDF-DevOps-Base.git

cd UDF-DevOps-Base/labs/lab0
```
### Set the BIG-IP password
```bash
export bigip_pwd=replaceme
```
### Onboard the two BIG-IP virtual appliances
```bash
COUNTER=1
for i in {6..7}
do  
    # authenticate to the BIG-IP
    docker exec -it f5-cli f5 login --authentication-provider bigip \
    --host 10.1.1.$i --user admin --password $bigip_pwd

    # POST the DO declaration
    docker exec -it f5-cli f5 bigip extension do create --declaration \
    /f5-cli/projects/UDF-DevOps-Base/labs/lab0/bigip$COUNTER.quickstart.do.json

    # increment counter
    let COUNTER=COUNTER+1
done
```
### Test that the BIG-IPs are ready
```bash
for i in {6..7}
do
    inspec exec do-ready -t ssh://admin@10.1.1.$i --password=$bigip_pwd \
    --input internal_sip=$10.1.10.$i external_sip=10.1.20.$i 
done
```
