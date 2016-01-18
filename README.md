# Installing SolrCloud For Titan 1.x
These instructions are for how to install a 2 node SolrCloud on Linux from scratch.
These instructions are the "quick" approach for setting up a SolrCloud and do not describe a production configuration. 
Zookeeper must be installed and available prior to starting the Solr installation. 
The steps assume a 2 node cluster, although this should work for a 3 and 4 node cluster as well  

1. Create a solr user on your Linux Operating System where this work will be done.  Do all steps as the solr user  
2. Make sure 'git' is installed and available to the solr user:    yum install git  
3. Oracle 8 package: jdk1.8.0_45 was used for this configuration  
4. Download solr from here:  
   http://mirror.olnevhost.net/pub/apache/lucene/solr/5.3.1/  
   http://mirror.olnevhost.net/pub/apache/lucene/solr/5.3.1/solr-5.3.1.tgz 
5. Get solr onto your machine  
   ```
   wget http://mirror.olnevhost.net/pub/apache/lucene/solr/5.3.1/solr-5.3.1.tgz 
   ``` 
 
6. gunzip solr-5.3.1.tgz  
7. tar -xvf solr-5.3.1.tar  
8. Change directories to the top level directory where solr was unzipped:

   ```
   cd solr-5.3.1
   ```  
9. Export the solr-5-3.1 directory as $SOLR_HOME  (or use this directory as solr home in the following instructions)  

10. Enter the following command to begin solr cloud set up and take ALL the defaults
 
   (Note that this will set up a sample SolrCloud with 2 virtual nodes on the SAME machine.  We will copy the  
   configuration to a second machine and adjust things to achieve a 2 node cluster on 2 machines)  

   bin/solr -e cloud  

   (select 2 for the number of nodes you'd like to run !)  
11. Enter the following command  
    ps -ef | grep solr  
   and verify that 2 solr instances are running on the single machine  
12. Stop both solr instances by using the kill command on their proc id  

   ```
   kill <proc-id>
   ```  
   Make sure they are both stopped  
13. tar the solr-5.3.1 subdirectory where solr is installed and where the 2 nodes were just created
   (You do not want to use the original 'install' tar image, but instead want to zip the directory
   with the config that we just created when creating the two node cluster)   
   tar -cvf solrnode.tar ./solr-5.3.1/*   
14. Copy the solrnode.tar file to the machine where your second SolrCloud node will run and untar it  
15. Next we do manual clean up of some unneeded files to create a true two node solr installation.  
   On the FIRST solr node (node1), navigate to $SOLR_HOME/example/cloud  
   You should see 2 directories called node1 and node2 in this directory to represent  
   the two 'virtual' nodes we created on a single machine.  Remove the node2 directory:  

   rm -rf node2  
   cd node1  
   rm -rf logs  
   mkdir logs  
   cd $SOLR_CLOUD/example/cloud/node1/solr  
   rm -rf gettingstarted*  
   rm -rf zoo_data  

   On the SECOND solr node (node2), navigate to the $SOLR_HOME/example/cloud directory  
   and remove the node1 directory:  

   rm -rf node1  
   cd node2  
   rm -rf logs  
   mkdir logs  
   cd $SOLR_CLOUD/example/cloud/node2/solr  
   rm -rf gettingstarted*  
   rm -rf zoo_data  
16. Next, we will clean up zookeeper entries created from creating the 2 node virtual
solr cloud with a fake collection  
  
   On the node where zookeeper is installed, go into the zookeeper CLI:  
   ./zkCli.sh  

   Issue the following commands in the ZooKeeper CLI:  

   [zk: localhost:2181(CONNECTED) 3] ls /  
   [configs, zookeeper, clusterstate.json, aliases.json, live_nodes, overseer, overseer_elect, collections]  
   [zk: localhost:2181(CONNECTED) 4] rmr /configs  
   [zk: localhost:2181(CONNECTED) 6] rmr /clusterstate.json  
   [zk: localhost:2181(CONNECTED) 7] rmr /aliases.json  
   [zk: localhost:2181(CONNECTED) 8] rmr /live_nodes  
   [zk: localhost:2181(CONNECTED) 9] rmr /overseer  
   [zk: localhost:2181(CONNECTED) 10] rmr /overseer_elect  
   [zk: localhost:2181(CONNECTED) 11] rmr /collections  
   [zk: localhost:2181(CONNECTED) 12] ls /  
   [zookeeper]  
   [zk: localhost:2181(CONNECTED) 13] quit  
   Quitting...  
17. Next, we want to copy the Titan solr configuration underneath Solr so it is available when we create Titan indexes that use Solr.  Use the directory name "titan" to match the commands that follow: 
  
   Download Titan 1.0 and build it, making sure to build the 1.0.0 branch. 
   mvn clean install -Dskiptests=true 
  
   Follow the instructions in chapter 32 of the Titan manual to copy the Titan sample SolrCloud configuration 
   underneath a configset directory on BOTH SolrCloud nodes in your [installation](http://s3.thinkaurelius.com/docs/titan/1.0.0/solr.html)  
18. Copy the jts-1.13.jar from the Titan lib directory, to the $SOLR_HOME/example/solr-webapp/webapp/WEB-INF/lib directory on both nodes  
19. Start both SolrCloud nodes:  

   On Node 1:  
   ```
   bin/solr start -c  -h <hostname> -p 8983 -d /home/solr/solr-5.3.1/server -z <zookeeper host>:2181 -m 512m  -s /home/solr/solr-5.3.1/example/cloud/node1/solr -V
   ```  

   On Node 2:  
   ```
   bin/solr start -c  -h <hostname> -p 8983 -d /home/solr/solr-5.3.1/server -z <zookeeper host>:2181 -m 512m  -s /home/solr/solr-5.3.1/example/cloud/node2/solr -V
   ```
20. Push the titan configuration set to ZooKeeper - adjusting paths and hostnames below for your configuration:  
   ```
   /usr/lib/jdk1.8.0_45/bin/java -Dsolr.install.dir=/home/solr/solr-5.3.1 -Dlog4j.configuration=file:/home/solr/solr-5.3.1/server/scripts/cloud-scripts/log4j.properties -classpath /home/solr/solr-5.3.1/server/solr-webapp/webapp/WEB-INF/lib/*:/home/solr/solr-5.3.1/server/lib/ext/* org.apache.solr.util.SolrCLI create_collection -name titan -shards 2 -replicationFactor 1 -confname titan -confdir titan  -configsetsDir /home/solr/solr-5.3.1/server/solr/configsets -solrUrl http://solrserver1.example.com:8983/solr
   ```
21. Open the SolrCloud console and verify the 2 node Solr Cloud is running:  

   ```
   http://<host name of either node in the solr cluster>:8983/solr
   ```  

   In the Solr console navigate to "Cloud", you should see something similar to this:  

   Navigate to "Cores" and you should see something similar to this  

22. You can go back to the zkCli and list the configuration set properties if you want:  
   [zk: localhost:2181(CONNECTED) 48] ls /config  
   [titan]  
  
   [zk: localhost:2181(CONNECTED) 49] ls /collections  
   [titan]  
23. When setting up the Titan 1.0 properties file, make sure to refer to the search index as "titan" and use the solr.configset property as described in chapter 23:  

   index.search.solr.configset=titan  

   (...most importantly is that the configset setting in the properties file for Titan matches what you pushed to ZooKeeper in SolrCloud)
