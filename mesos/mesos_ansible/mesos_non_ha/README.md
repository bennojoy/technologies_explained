## Ansible playbook for deploying a non HA mesos cluster on AWS.

To run this playbook you would need ansible core to be installed on your control host. Steps to install Ansible is available in the below link.

    http://docs.ansible.com/ansible/intro_installation.html

Once installed checkout this repo

    git clone https://github.com/bennojoy/technologies_explained.git

change directory to the new repo 

    cd technologies_explained/mesos/mesos_non_ha

Set you aws credentials:

    export AWS_ACCESS_KEY='foo'
    export AWS_SECRET_KEY='bar'

Set your environement specific aws variables in group_vars/all. especially the below ones.

    #The vpc and and subnet where the instances should be created.
    vpc_id: vpc-7c19
    vpc_subnet_id: subnet-983ec
    keypair: benkey 
    #The number if slave instances required.
    mesos_slave_count: 2 

Once done run the playbook
    
    ansible-playbook -i hosts mesos_non_ha_aws.yml -vv

This should setup the Mesos cluster for you. The last line at the end would show you the ip of the master server. 
You can see the cluster status via browsing:

    http://<mesos_master>:5050

You can also test it via our example factorial framwork, by running:

    factorial_schedular 5 7

it should return the factorial of 5 and 7.


