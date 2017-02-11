# Working Zeta Step By Step
----------

### Initial Prep and Instance launch (AWS)
----------

**Note on choice of AWS:** Zeta can be deployed to other clouds and we can even use APIs for this later

1. Get an Amazon AWS Account. Upload a keypair (without a password) so you have one accessible to use for your cluster. 
2. Spin up 5 hosts (I used `m3.xlarge`) running Ubuntu 16.04 (For now)
    - Note: Nodes must have at least 16GB of ram, and at least one extra disk (other than OS disk) for MapR Services to install.
3. Note the subnet (Mine was `172.31.0.0/16`) - If you are unclear here, you may need to create your own subnet for testing. All nodes, but be able to talk to all nodes (via security policy) on all ports
4. On the Add storage, the two disks is correct, but up up the root volume to be 50 GB instead of 8
5. Security Group - SSH (`22`) from anywhere, All traffic from your subnet (mine was `172.31.0.0/16`) and All traffic from your IP (we will change this later, once we get the node firewalls up, we will open this to world to keep things easier to manage)
6. Launch with the private key you intend to use from the initial node prep in the [Prep Cluster](#prep-cluster) instructions below
7. Pick a node, upload the private key from step 1 and connect to this node. 
8. Go to [Clone Repository](#clone-repository)

### Initial Prep and Instance launch (Google Cloud Platform)
----------

1. Get a Google Cloud Platform account. Upload a keypair (without a password) so you have one accessible to use for your cluster. 
2. Create an instance group for 5 instances. 
    - For "Instance Template", click create new template, and create an instance template with Ubuntu 16.04 LTS, Machine Type of 4vCPUs (15gb memory - n1-standard-4) and 50gb of SSD storage.
    - Create the instance group, which will automatically create 5 instances for you.
3. Go to the Metadata Page and click SSH keys. Add your local machine's pub key here. This will let you log in to all isntances in the project. (https://console.cloud.google.com/compute/metadata/sshKeys)
4. Clcik "VM Instances" on the left sidebar to see your node IPs. These will be needed in the [Prep Cluster](#prep-cluster) section.
5. For each Instance, click the instance name, and click "Edit" at the top. You will need to attach a 10gb (google won't let you use less than 10gb per disk) disk to each node for Mapr to use.
6. Click "Add item" under Additional disks, and create a disk with 10gb of space and no image. Repeat this for all nodes on the cluster.
7. Now we need to reboot and ssh into the nodes and mount these disks. I found that a few incorrect attempts locked me out, requiring a node reboot in the cloud console, so if ssh hangs, reboot the node. Alternatively, you can click "SSH" on the VM Instances page to open a browser window with an ssh client active.
    - Check out the mounting guide here to learn how to mount a disk in linux: https://cloud.google.com/compute/docs/disks/add-persistent-disk
    - You will need to do this on each node. I've included a bash script in scripts/googleCloud/storageMount that will do this for you. It should look like:
    ```
    //Get the disk ID (I've named mine google-zeta-storage-disk-x, where x is a number 1-5 for a node that the disk belongs to) 
    ls /dev/disk/by-id
    
    //Format your new storage disk
    sudo mkfs.ext4 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/disk/by-id/google-zeta-storage-disk-1 
    
    //Make the mounting directory
    sudo mkdir -p /mnt/disks/storage
    
    //Mount it
    sudo mount -o discard,defaults /dev/disk/by-id/google-zeta-storage-disk-1 /mnt/disks/storage

    //Give permissions (read/write) to the device for all users
    sudo chmod a+w /mnt/disks/storage
    
    //Reattach on boot
    echo UUID=`sudo blkid -s UUID -o value /dev/disk/by-id/google-zeta-storage-disk-1` /mnt/disks/storage ext4 discard,defaults,nofail 0 2 | sudo tee -a /etc/fstab
    ```
8. Go to [Clone Repository](#clone-repository)

### Initial Prep - On Prem
----------
1. Get some master nodes
    - No extra disk needed. min 32 GB of ram, rec 64. A 3 node quorum of masters running 64 GB of ram can scale to the 10s of thousands of agent nodes
    - A good amount of disk space - at least a TB 
    - 1U Units work well. 
2. Get some agent nodes
    - Since we are doing storage, we like ensuring good specs. 
        - At least 2 Octo-Core (with hyperthreading) processors
        - 256 GB or more of Ram
        - 2 main (OS) harddrives with over 750GB available. Fast is good here (SSD, 15k) running in RAID1
        - 8 or more "Storage" drives. Faster equates to better storage in your cluster
        - Networking: We like upping this as much as possible. 4 10Gbps Copper, bonded into two HA pairs.  One for the main Zeta (Routable) subnet and another for fast disk backchannel
    - Aim for at least 5 nodes to start with. 
3. You must go to each node and have an account that is accessible by a SSH keypair (**with no password**), and has passwordless sudo rights. After install, you can deprivilege this account.  Go to [Clone Repository](#clone-repository)

### Notes: Initial Prep - Edge Nodes vs. Public 
----------
Public nodes in DCOS run with the "slave_public" role. That means taks will not run on them unless explicitly told to. This works great when you want to have a public node that is your edge node, and provides a hop point into your cluster than you can control 

For some clusters, having a "smaller" edge node in the slave_public role to be this hop point is a great idea. In AWS for example, adding a node specifically for this is easy. 

If you have a on-prem cluster where having a large node dedicated to a special role for edge services is wasteful, and you rely on other security measures in your enterprise. You can still have an "edge" node for services, but it will run standard task in addition to the load balancing functions. 

That is the fundamental difference, if you don't want to have "public" nodes as defined by DCOS, then leave the public prompt in the DCOS config creation blank, and instead specify edge nodes manually in the network configuration.  If you specify public nodes in the DCOS config, it will auto populate the suggested node in the network config. 


### Clone Repository
----------

**Note:** this prep will not work on MacOS.  Please run from a Linux host to start (See [AWS](#initial-prep-and-instance-launch-aws) note about start node)

1. `$ git clone https://github.com/JohnOmernik/zetago`
    - **Note:** Ensure your priv key you used to start the nodes is on the box you are doing the prep from.  
2. `$ cd zetago`

### Prep Cluster
----------
1. `$ ./zeta prep`
    - This is the initial prep configuration creator. It will ask some quesstions, defaults are great for most things unless noted here:
    - The Initial user list is a list of users the scripts try to connect to the nodes.  If you need to add to the default list just make is space separated
    - the Key is the key that is used to connect to the instance the first time. This will change after we prep the nodes. 
    - Passwords for the `zetaadm` (IUSER) and `mapr` users. Pretty clean cut. 
    - Space separated list of nodes to connect to. These are the IPs that the scripts will connect to initially to do the setup. If on prem, they may be internal IPs, if AWS they may be external depending on where you prepping from.
      - On AWS if you are connecting to a node and doing all prep from there, use the internal AWS IPs. If you are using an off AWS Network linux box to do the initial prep, use the External IPs to AWS.
    - Initial node is the node that will be used to complete the zeta install. just pick one of the nodes in the previous list
    - Interface check list: This is just the list and order of interfaces we check for "internal" IPs. This should be ok to leave as is, unless you want to add or reorder for your env
    - All conf info is in `./conf/prep.conf`
2. At this point the prep.conf is written, but you should review it you can do this by:
    - `$ ./zeta prep` # And then press `E` to examine the conf
    - If this looks good go ahead press `U` to use the conf. Then enter:
    - `$ ./zeta prep -l` # This locks the conf, so it doesn't prompt you every time. It asks you to use the prep.conf again and then confirms the locking. Now it will assume the conf is correct. 
3. Your conf is created, reviewed, and locked, the next step is to prep the nodes.
    - $ `./zeta prep install -a -u` # This installs the prep on all nodes (`-a`) and does so unattended (`-u`) however it will prompt you for the version file to use... (Just use the default 1.0.0 by hitting enter)
    - If you are running from one of the nodes, then just wait until your connection drops. This will be your node rebooting. When complete, connect back to the same node with your initial user. From here, you should connect to your initial node (You should be able to get to that node by cd zetago; `./zeta prep` # It wil give you instructions on which node you picked and how to connect. 
4. Once this completes, you can check the status of the nodes by running `$ ./zeta prep status`  Once all nodes are running with Docker, you can proceed to the next steps. 
5. Also, to get the SSH command to connect to the initial nodes, type `./zeta prep` at anytime

### Connect to cluster
----------
1. Once the `./zeta prep status` command returns docker installed on all nodes it's time to switch the context. Instead of running commands using ./zeta from your machine and user, you will be connecting to the initial node (specidfied in the config) and running all remaining commands from there
    - You can get the exact SSH command to connect to the initial node by typing ./zeta prep on your machine, it will show you the SSH command to connect to the initial node with zetaadm (IUSER). 
2. To get the SSH command to use, just type `./zeta prep` and it will show you the SSH command to connect to the initial node
3. Once connected to the inital node run `$ cd zetago`  # All commands will start from here

### DCOS and Firewall Install
----------
1. Run the prep by typeing `$ ./zeta dcos`
    - It will ask some questions about conf, including showing you the internal IPs of your nodes
    - IP for bootstrap node (use the init node, the default)
    - Port for bootstrap (use the default)
    - Pick one of the remaining (non-bootstrap) Internal IPs to be the master node (or if you doing lots of nodes, pick multiple, space separated)
    - Now you have the option to pick (a) public node(s).  This will be your edge node for network as well, and only run tasks specified as slave_public.  You may leave this blank, 
    - clustername: pick anything
    - dns resolvers should be setup for AWS, but if you are on another platform, please correct them
    - Proxy information is required if you are behind a proxy. 
    - All conf info is in `./conf/dcos.conf`
2. Prior to doing the DCOS install, it's now time to run the network setup: Run `$ ./zeta network`
    - You cannot start further DCOS install processes without putting a firewall up. 
    - This command sets the network.conf file with information about your networking setup 
        - NTP Servers: If you specify this, it will limit outbound NTP request only to those servers, otherwise, if left blank, we allow outbound NTP to all.
        - Zeta Routable Addresses: This is your main subnet, the subnet that your default gateway is on that all servers connect to. For me, on AWS, it's `172.31.0.0/16`, but it may be different for you. 
        - Zeta Non-Routable Addresses: If you have other interfaces that connect all nodes (non-gateway interfaces)  You can add them here to ensure node to node communication is unfettered. So lets say your routable interfaces are `192.168.0.0/24`.  Then you have two more network interfaces between nodes that are not routable: `10.0.4.0/24` and `192.168.250.0/24` Then you would put here: `10.0.4.0/24,192.168.250.0/24`  If you have no additional interfaces, just leave blank
        - Static IPs of Remote Machines: If you have certain machines that you will bedoing administration from, put them here comma separated. 
        - List of remote admin machine names: If you are on a network that is dynamic, and wish to use a reverse lookup for admin machines, you can enter names here. The FW script will attempt to look up each name, and add the look up IP to the remote allowed IP list. 
        - If you have an unstable (IP Wise) connection, you may want to always allow port `22` connections from the world. The default is `N`, but if you want to allows this, set to `Y`. 
        - Allow direct outbound connections on `80/443`: If you don't use a proxy, then you must say yes here, if not, your zeta will fail. If you use a proxy, and specified in the DCOS config, you can say no here. 
        - Edge nodes: These are the nodes that will allow services into your cluster. It will auto populate with public nodes from the DCOS config, but you can add more nodes, or use non-public nodes (if you don't have public nodes) here.  
3.  Now that the initial config is setup, install the firewall on each node by running `./zeta network deployfw -f="Initial firewall deployment"`
    - The flag `-f="Nodes on deploy"` are the nodes that each deploy uses in the change log. 
    - This updates the `FW_FIRST_RUN` flag in the `network.conf` to allow you to continue to install DCOS. 
    - It's recommened at this point if you had a more restrictive AWS Security group to now open things up wide open for the cluster. We are managing the firewall connnections now. 
4. Start the bootstrap server by running `$ ./zeta dcos bootstrap`
5. Install DCOS by running `$ ./zeta dcos install -a`
    - This installs DCOS first on the Masters specified, then on the remaining nodes. 
    - It also provides the Master UI Address (as well as exhibitor)
    - The Master will take some time to come up, it checks this, and doesn't install agents until after the master is healthy.
6. Check the UI for DCOS and once all your agents are connected, rerun the firewall now that DCOS is deployed. `$ ./zeta network deployfw -f="Post DCOS Firewall Deploy"`
    - Note, if you are on an unstable IP and instead of specifying remote admin machines, you instead opted for open to the world SSH. UIs are a bit tricky.
    - One option is to connect via ssh and use a remote socks proxy. `ssh -D8080 user@clusterip` to connect then set your browser to use a SOCKS proxy at -d8080.
    - The other option is to just add your IP to the `./conf/network.conf` file and redeploy the firewall. 

### Filesystem Install
----------
1. Once DCOS is is up and running and the Agents properly appear in the UI its time for a Shared Filesystem
2. First create the conf file by running `./zeta fs`
    - Pick a Provider. The only provider have right now is mapr, so I would choose that. (Otherwise nothing will work)
    - It will ask you about your FS Docker registery. Defaults are best here. 
    - Once this is complete, it will automatically invoke the fs_$PROVIDER create conf and ask you questions specific to your provider 
    - FS conf is written to `./conf/fs.conf`
    - FS_$PROVIDER conf is written to ./conf/fs_$PROVIDER.conf (for example) `./conf/fs_mapr.conf`
    - The initial cluster conf is also invoked here. There are no questions to answer, but this conf is written to `./conf/cluster.conf`
3. Mapr Provider Questions:
    - Defaults work for most things, here are some notes:
    - MapR installation directory should be default. (`/opt/maprdocker`)
    - ZK string is the first interesting one. You need to determine how many agents will run ZK instances. In a small cluster 1 is ok, 3 is better, 5 is best. 
        - The ZK string format should be `NodeID:Hostname,NodeID:Hostname`
        - The script prints out the internal host names) so if you have one, then it's just Node ID 0, if two then Node ID 0 and Node ID 1
        - **Example:** `0:node.aws.local,1:node2.aws.local`
        - ZK ports should be left as defaults
    - CLDB String is pretty simple, pick one (or more) servers for a CLDB (Master MapR Node) and put the hostname colon port (7222) `node1:7222`
    - The initial disks string is important.
        - The format is `NODE1:/dev/disk1,/dev/disk2;NODE2:/dev/disk1,/dev/disk2` (disk 1 may be sda, disk 2 may be sdb, each system may be different)
        - The disks should be unformatted block devices. 
        - THat said, often times when you are starting a cluster, each node DOES have the same disks for MapR's use. The script asks that, if it's the case, hit enter to enter a disk string once and apply it to all nodes
        - For example, on the m3.xlarge nodes I use, I can use `/dev/xvdb,/dev/xvdc` as my disk string and apply it to all nodes. 
    - NFS Nodes
        - Nodes to install NFS Server on the nodes
        - This is not highly tested at this time
    - MapR Subnet:
        - These are the subnets of the interfaces to use for Node to Node Communications
        - It will read the data from the FW Config and use that
        - I use `172.31.0.0/16` because that's the range AWS puts me in... enter something here that works for your env
    - Use the default vers file
    - All conf information is `./conf/fs_mapr.conf`
3. Install local zetaca via `$ ./zeta cluster zetaca install`
    - Choose both the certificate auth defaults and the certificate defaults for your cluster here
    - You will need to enter a password to bootstrap the CA, this is the key for your CA
    - This runs locally for now (as there is no shared filesystem). 
4. Install the fs docker registry via `$ ./zeta fs fsdocker`
    - This runs locally for now, as there is no shared filesystem
5. At this point we are ready to build our ZK and MapR containers. 
    - `$ ./zeta fs mapr buildzk -u` Build the ZK container
    - `$ ./zeta fs mapr buildmapr -u` Build the mapr container
6. Once built, we need to start the ZKs first: 
    - `$ ./zeta fs mapr installzk -u`
7. Once ZKs are running start them MapR containers:
    - `$ ./zeta fs mapr installmapr -u`
    - It will start CLDBs first, wait until they are up and running, and then run the standard nodes
    - Once it submits the standard nodes, it will provide a CLDB link. You can log into the MapR Console using the FSUSER password you provided in prep for what is likely the `mapr` user. 
8. Now to complete MapR/FS install by installing fuse clients on nodes. 
    - `$ ./zeta fs mapr installfuse -u` This installs the FUSE client on all agents so it's ready for cluster installation (ZETA!)
10. Create some local volumes for use in MapReduce shuffle activities
    - `$ ./zeta fs mapr createlocalvols -a -u`

### Cluster (Zeta) Install
----------
1. The `cluster.conf` file was already created (the FS install calls the cluster install script secretly)
2. The first step is to install the zeta base directories. This will clear your cluster directories in your initial install. 
    - `$ ./zeta cluster zetabase`
3. Next we will be installing the shared docker registry that is backed by MapR (instead of local)
    - `$ ./zeta cluster shareddocker`
4. Authentication! We now install the shared OpenLDAP server
    - `$ ./zeta cluster sharedldap`
    - Note: It will warn that the CA is not yet moved to the final location. It will ask you to do this, `Y` (the default) is the correct answer here
    - Enter a ldapadmin password (this is the admin account for account administration)
5. Now we have to setup ldap auth on all hosts so we are all aware of the connections
    - `$ ./zeta cluster allldaphosts`
6. We need a shared role. Packages rely on it, the cluster needs it. Install it!
    - `$ ./zeta cluster addsharedrole`
    - It will ask for a password for the share service user the default UID start of `1000000` is good here too
7. Install the ldapadmin web UI
    - `$ ./zeta cluster sharedldapadmin`
8. The final step in this part is to install some performance modifications to the ldap server. It does cause a ldap server reboot, but its worth it
    - `$ ./zeta cluster ldapperf -a -u`
9. (Optional) I like to add a user role now (say "prod")
    - `$ ./zeta cluster addzetarole -r=prod`
    - Manually increment the UID from the provided `1000000` (which is shared) 1 million to `2000000` for each role you add. # TODO: Automate this... 

### Users and Groups
----------
1. Users and groups can be created in the ldapadmin service
2. They can also be created on the command line using `./zeta users` * (More documentation and updates coming soon)

### Packages 
----------
1. The best part about zeta is the packages!
2. You can build, install, start, stop, and uninstall pacakges with the `./zeta package` interface
3. `$ ./zeta package` # For more info - This also builds the `package.conf` on first run. 
4. A Very important package if you want to do services is the marathonlb package `./zeta package build marathonlb` and `./zeta package install marathonlb`
    - A good example to start with is Apache Drill
    - Build: `./zeta package build drill`
    - Install `./zeta package install drill`
    - Start: Following the command provided by `./zeta package install drill` to start your drill instance!


### Adding Nodes # TODO
----------
1.  Run: `./zeta dcos sshhosts` # `-u` if you want unattended
    - The Create conf for dcos does ask you to do this, so you don't have to do it manually
    - If you run this command, it uses ssh-key to get the host key of all nodes and trust them by adding it to the known_hosts file. It does this for the IP, the short name, and the fully qualified name of every host. 
    - You can run this with `-u` if you don't want to it to prompt. Otherwise it will prompt once before trusting all keys. 
