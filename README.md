Shell

Run HBase CLI commands:

$ hbase 
$ hbase hbck 
$ hbase shell 

Enable command history:

$ vi ~/.irbrc
$ cat ~/.irbrc
require 'irb/ext/save-history'
IRB.conf[:SAVE_HISTORY] = 100
IRB.conf[:HISTORY_FILE] = "#{ENV['HOME']}/.irb-save-history"
Kernel.at_exit do
  IRB.conf[:AT_EXIT].each do |i|
    i.call
  end 
end

Optionally: Restart services when you have suspended the VM and the services do not run anymore (can happen):

$ sudo service hbase-master start
starting master, logging to /var/log/hbase/hbase-hbase-master-quickstart.cloudera.out
Started HBase master daemon (hbase-master):                [  OK  ]
$ sudo service hbase-regionserver start
Starting Hadoop HBase regionserver daemon: starting regionserver, logging to /var/log/hbase/hbase-hbase-regionserver-quickstart.cloudera.out
hbase-regionserver.

Start shell, and run some commands:

$ hbase shell
2015-11-25 02:16:29,575 INFO  [main] Configuration.deprecation: hadoop.native.lib is deprecated. Instead, use io.native.lib.available
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.0.0-cdh5.4.2, rUnknown, Tue May 19 17:07:29 PDT 2015

hbase(main):001:0> list
TABLE                                                                                                                                                  
0 row(s) in 0.3010 seconds

=> []
hbase(main):002:0> version
1.0.0-cdh5.4.2, rUnknown, Tue May 19 17:07:29 PDT 2015

hbase(main):003:0> status
1 servers, 0 dead, 2.0000 average load

Create a simple table, add some data and retrieve it:

hbase(main):004:0> create 'testtable', 'cf1'
0 row(s) in 0.6880 seconds

=> Hbase::Table - testtable
hbase(main):005:0> put 'testtable', 'row-1', 'cf1:col-1', 'val-1'
0 row(s) in 0.1570 seconds

hbase(main):006:0> get 'testtable', 'row-1'
COLUMN                                 CELL                                                                                                            
 cf1:col-1                             timestamp=1448446633260, value=val-1                                                                            
1 row(s) in 0.0260 seconds

Create another table, with two column families, also giving it a description using the arbitrary metadata storage available:

hbase(main):008:0> create 'testtable2','cf1', 'cf2', METADATA => { 'Description' => 'Test table for training' }
0 row(s) in 0.4250 seconds

=> Hbase::Table - testtable2
hbase(main):009:0> list
TABLE                                                                                                                                                  
testtable                                                                                                                                              
testtable2                                                                                                                                             
2 row(s) in 0.0080 seconds

=> ["testtable", "testtable2"]

hbase(main):011:0> describe 'testtable2'
Table testtable2 is ENABLED                                                                                                                            
testtable2, {TABLE_ATTRIBUTES => {METADATA => {'Description' => 'Test table for training'}}                                                            
COLUMN FAMILIES DESCRIPTION                                                                                                                            
{NAME => 'cf1', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', VERSIONS => '1', COMPRESSION => 'NONE', MIN_VERSIONS => 
'0', TTL => 'FOREVER', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                                
{NAME => 'cf2', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', VERSIONS => '1', COMPRESSION => 'NONE', MIN_VERSIONS => 
'0', TTL => 'FOREVER', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                                
2 row(s) in 0.0360 seconds

Add data using a Ruby loop:

hbase(main):012:0> for i in 1..100 do put 'testtable', "row-3{i}", 'cf1:col-1', "val-#{i}" end 
0 row(s) in 0.0100 seconds

0 row(s) in 0.0080 seconds

...

0 row(s) in 0.0050 seconds

0 row(s) in 0.0070 seconds

=> 1..100

Access rows directly using “get”:

hbase(main):013:0> get 'testtable', 'row-1'
COLUMN                                 CELL                                                                                                            
 cf1:col-1                             timestamp=1448446633260, value=val-1                                                                            
1 row(s) in 0.0160 seconds

hbase(main):014:0> get 'testtable', 'row-2'
COLUMN                                 CELL                                                                                                            
0 row(s) in 0.0040 seconds

Using a “scan” to retrieve more rows, here between a start and end row:

hbase(main):015:0> scan 'testtable', { STARTROW => 'row-5', STOPROW => 'row-15' }
ROW                                    COLUMN+CELL                                                                                                     
0 row(s) in 0.0340 seconds

Question: Why is there nothing returned? Let’s see what is inside using a scan with a limit:

hbase(main):016:0> scan 'testtable', LIMIT => 3
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448446633260, value=val-1                                                          
 row-3{i}                              column=cf1:col-1, timestamp=1448446823246, value=val-100                                                        
2 row(s) in 0.0210 seconds

Apparently the script was wrong, there was a “3” in there instead of a “#”. Truncating the table and reinserting the data:

hbase(main):018:0> truncate 'testtable'
Truncating 'testtable' table (it may take a while):
 - Disabling table...
 - Truncating table...
0 row(s) in 1.5230 seconds

hbase(main):019:0> for i in 1..100 do put 'testtable', "row-#{i}", 'cf1:col-1', "val-#{i}" end 
0 row(s) in 0.1340 seconds

0 row(s) in 0.0080 seconds

...

0 row(s) in 0.0090 seconds

0 row(s) in 0.0060 seconds

=> 1..100

Trying the scan again, but… still nothing:

hbase(main):020:0> scan 'testtable', { STARTROW => 'row-5', STOPROW => 'row-15' }
ROW                                    COLUMN+CELL                                                                                                     
0 row(s) in 0.0110 seconds

Verifying the content again, we realize that the lexicographical sorting means we cannot use “row-5” as start, and "row-15” as the stop row parameter, because the are sorted the other way around (since the “1” in “row-15” is less than the “5” in “row-5” comparing the keys byte by byte):

hbase(main):022:0> scan 'testtable', LIMIT => 20
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448446949474, value=val-1                                                          
 row-10                                column=cf1:col-1, timestamp=1448446949549, value=val-10                                                         
 row-100                               column=cf1:col-1, timestamp=1448446950196, value=val-100                                                        
 row-11                                column=cf1:col-1, timestamp=1448446949556, value=val-11                                                         
 row-12                                column=cf1:col-1, timestamp=1448446949561, value=val-12                                                         
 row-13                                column=cf1:col-1, timestamp=1448446949570, value=val-13                                                         
 row-14                                column=cf1:col-1, timestamp=1448446949576, value=val-14                                                         
 row-15                                column=cf1:col-1, timestamp=1448446949584, value=val-15                                                         
 row-16                                column=cf1:col-1, timestamp=1448446949594, value=val-16                                                         
 row-17                                column=cf1:col-1, timestamp=1448446949601, value=val-17                                                         
 row-18                                column=cf1:col-1, timestamp=1448446949607, value=val-18                                                         
 row-19                                column=cf1:col-1, timestamp=1448446949614, value=val-19                                                         
 row-2                                 column=cf1:col-1, timestamp=1448446949491, value=val-2                                                          
 row-20                                column=cf1:col-1, timestamp=1448446949621, value=val-20                                                         
 row-21                                column=cf1:col-1, timestamp=1448446949627, value=val-21                                                         
 row-22                                column=cf1:col-1, timestamp=1448446949634, value=val-22                                                         
 row-23                                column=cf1:col-1, timestamp=1448446949642, value=val-23                                                         
 row-24                                column=cf1:col-1, timestamp=1448446949650, value=val-24                                                         
 row-25                                column=cf1:col-1, timestamp=1448446949657, value=val-25                                                         
 row-26                                column=cf1:col-1, timestamp=1448446949663, value=val-26                                                         
20 row(s) in 0.0750 seconds

To fix the scan, we have to use proper start and stop row keys, so that they really sort one after the other:

hbase(main):024:0> scan 'testtable', { STARTROW => 'row-5', STOPROW => 'row-55' }
ROW                                    COLUMN+CELL                                                                                                     
 row-5                                 column=cf1:col-1, timestamp=1448446949513, value=val-5                                                          
 row-50                                column=cf1:col-1, timestamp=1448446949828, value=val-50                                                         
 row-51                                column=cf1:col-1, timestamp=1448446949835, value=val-51                                                         
 row-52                                column=cf1:col-1, timestamp=1448446949842, value=val-52                                                         
 row-53                                column=cf1:col-1, timestamp=1448446949850, value=val-53                                                         
 row-54                                column=cf1:col-1, timestamp=1448446949856, value=val-54                                                         
6 row(s) in 0.0390 seconds

Let’s now create a namespace, and a more specific table, setting the stored versions higher (from the default of 1) and enabling Snappy compression to:

hbase(main):025:0> create_namespace 'development'
0 row(s) in 0.0630 seconds

hbase(main):026:0> create 'development:staging-src1', { NAME => 'cf1', VERSIONS => 3, COMPRESSION => 'SNAPPY' }, 'cf2' 
0 row(s) in 0.5190 seconds

The Ruby shell allows to store the table reference in a variable for reuse. Using the variable or the above fully specified shell commands is (well, should be) synonymous:

hbase(main):030:0> t = get_table 'development:staging-src1'
0 row(s) in 0.0000 seconds

=> Hbase::Table - development:staging-src1

hbase(main):031:0> t.put 'row-1', 'cf1:col-1', 'val-1'
0 row(s) in 0.0130 seconds

hbase(main):032:0> t.get 'row-1'
COLUMN                                 CELL                                                                                                            
 cf1:col-1                             timestamp=1448447591017, value=val-1                                                                            
1 row(s) in 0.0150 seconds

hbase(main):033:0> t.put 'row-1', 'cf2:col-1', 'val-1'
0 row(s) in 0.0250 seconds

hbase(main):034:0> t.put 'row-2', 'cf1:col-2', 'val-2'
0 row(s) in 0.0060 seconds

hbase(main):035:0> t.scan
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0250 seconds

hbase(main):036:0> scan 'development:staging-src1'
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0210 seconds

hbase(main):038:0> t.get 'row-1'
COLUMN                                 CELL                                                                                                            
 cf1:col-1                             timestamp=1448447591017, value=val-1                                                                            
 cf2:col-1                             timestamp=1448447614823, value=val-1                                                                            
2 row(s) in 0.0120 seconds

hbase(main):039:0> t.get 'row-1', 'cf1'
COLUMN                                 CELL                                                                                                            
 cf1:col-1                             timestamp=1448447591017, value=val-1                                                                            
1 row(s) in 0.0130 seconds

hbase(main):040:0> t.put 'row-1', 'cf1:col-2', 'val-2'
0 row(s) in 0.0070 seconds

hbase(main):041:0> t.scan
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0220 seconds

hbase(main):042:0> t.get 'row-1', 'cf1'
COLUMN                                 CELL                                                                                                            
 cf1:col-1                             timestamp=1448447591017, value=val-1                                                                            
 cf1:col-2                             timestamp=1448447750441, value=val-2                                                                            
2 row(s) in 0.0070 seconds

hbase(main):043:0> t.get 'row-1', 'cf1:col-2'
COLUMN                                 CELL                                                                                                            
 cf1:col-2                             timestamp=1448447750441
1 row(s) in 0.0280 seconds

Using the JRuby way of creating a Java Date instance, we can decipher the epoch by converting it into a human-readable date:

hbase(main):044:0> java.util.Date.new(1448447750441)
=> #<Java::JavaUtil::Date:0x22ffc505>

hbase(main):045:0> java.util.Date.new(1448447750441).to_string
=> "Wed Nov 25 02:35:50 PST 2015"

Next we snapshot the table as-is:

hbase(main):046:0> snapshot 'development:staging-src1', 'devstagingsrc1-before-qa'
0 row(s) in 0.7060 seconds

hbase(main):047:0> scan 'development:staging-src1'
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0330 seconds

hbase(main):048:0> list_snapshots
SNAPSHOT                               TABLE + CREATION TIME                                                                                           
 devstagingsrc1-before-qa              development:staging-src1 (Wed Nov 25 02:39:29 -0800 2015)                                                       
2 row(s) in 0.0550 seconds

=> ["devstagingsrc1-before-qa"]

Let us now modify the original table, by deleting a row (note we are using “deleteall”, which applies to a row, since “delete” applies to a column instead):

hbase(main):051:0> deleteall 'development:staging-src1', 'row-1'
0 row(s) in 0.0290 seconds

hbase(main):052:0> scan 'development:staging-src1'
ROW                                    COLUMN+CELL                                                                                                     
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
1 row(s) in 0.0130 seconds

We restore the snapshot by cloning it into a new table:

hbase(main):055:0> clone_snapshot 'devstagingsrc1-before-qa', 'development:staging-src1-clone'
0 row(s) in 0.4930 seconds

hbase(main):056:0> list
TABLE                                                                                                                                                  
development:staging-src1                                                                                                                               
development:staging-src1-clone                                                                                                                         
testtable                                                                                                                                              
testtable2                                                                                                                                             
4 row(s) in 0.0170 seconds

=> ["development:staging-src1", "development:staging-src1-clone", "testtable", "testtable2"]

hbase(main):057:0> scan 'development:staging-src1-clone'
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0300 seconds

All the data is back, but in a new table. To restore the old table in place, we first need to disable it, then invoke the restore command:

hbase(main):058:0> disable 'development:staging-src1'
0 row(s) in 1.2820 seconds

hbase(main):059:0> restore_snapshot 'devstagingsrc1-before-qa'
0 row(s) in 0.8030 seconds

hbase(main):060:0> list
TABLE                                                                                                                                                  
development:staging-src1                                                                                                                               
development:staging-src1-clone                                                                                                                         
testtable                                                                                                                                              
testtable2                                                                                                                                             
4 row(s) in 0.0130 seconds

=> ["development:staging-src1", "development:staging-src1-clone", "testtable", "testtable2"]

hbase(main):061:0> describe 'development:staging-src1'
Table development:staging-src1 is DISABLED                                                                                                             
development:staging-src1                                                                                                                               
COLUMN FAMILIES DESCRIPTION                                                                                                                            
{NAME => 'cf1', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'SNAPPY', VERSIONS => '3', TTL => 'FOREVE
R', MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                              
{NAME => 'cf2', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'NONE', VERSIONS => '1', TTL => 'FOREVER'
, MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                                
2 row(s) in 0.0320 seconds

hbase(main):062:0> enable 'development:staging-src1'
0 row(s) in 0.4690 seconds

hbase(main):063:0> describe 'development:staging-src1'
Table development:staging-src1 is ENABLED                                                                                                              
development:staging-src1                                                                                                                               
COLUMN FAMILIES DESCRIPTION                                                                                                                            
{NAME => 'cf1', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'SNAPPY', VERSIONS => '3', TTL => 'FOREVE
R', MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                              
{NAME => 'cf2', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'NONE', VERSIONS => '1', TTL => 'FOREVER'
, MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                                
2 row(s) in 0.0330 seconds

hbase(main):064:0> scan 'development:staging-src1'
ROW                                    COLUMN+CELL                                                                                                     
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0260 seconds

Snapshots can be deleted in bulk like so:

hbase(main):065:0>
hbase(main):065:0> delete_all_snapshot '.*'
SNAPSHOT                               TABLE + CREATION TIME                                                                                          
 devstagingsrc1-before-qa              development:staging-src1 (Wed Nov 25 02:39:29 -0800 2015)                                                      

Delete the above 1 snapshots (y/n)?
y
0 row(s) in 0.0480 seconds
1 snapshots successfully deleted.

hbase(main):066:0> list_snapshots
SNAPSHOT                               TABLE + CREATION TIME                                                                                          
0 row(s) in 0.0080 seconds

=> []

hbase(main):067:0> list
TABLE                                                                                                                                                  
development:staging-src1                                                                                                                              
development:staging-src1-clone                                                                                                                        
testtable                                                                                                                                              
testtable2                                                                                                                                            
4 row(s) in 0.0190 seconds

=> ["development:staging-src1", "development:staging-src1-clone", "testtable", "testtable2"]

We resume with the table from earlier using the reference variable (just because):

hbase(main):068:0> t.describe
Table development:staging-src1 is ENABLED                                                                                                              
development:staging-src1                                                                                                                              
COLUMN FAMILIES DESCRIPTION                                                                                                                            
{NAME => 'cf1', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'SNAPPY', VERSIONS => '3', TTL => 'FOREVE
R', MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                              
{NAME => 'cf2', DATA_BLOCK_ENCODING => 'NONE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'NONE', VERSIONS => '1', TTL => 'FOREVER'
, MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}                                
2 row(s) in 0.0400 seconds

We put a new value into an existing column to test the versioning that HBase supports:

hbase(main):069:0> t.put 'row-1', 'cf1:col-1', 'val-99'
0 row(s) in 0.0090 seconds

The scan shows the current content of the table, which is the default setting:

hbase(main):070:0> t.scan
ROW                                    COLUMN+CELL                                                                                                    
 row-1                                 column=cf1:col-1, timestamp=1448448598740, value=val-99                                                        
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0250 seconds

You can override the versions to returned per scan, and also reverse the output. Note how using the variable does not work (though it should):

hbase(main):072:0> t.scan, { REVERSED => true }
SyntaxError: (hbase):73: syntax error, unexpected end-of-file

hbase(main):074:0> t.scan, { VERSIONS => 10 }
SyntaxError: (hbase):75: syntax error, unexpected end-of-file

hbase(main):077:0> scan 'development:staging-src1', { VERSIONS => 10 }
ROW                                    COLUMN+CELL                                                                                                    
 row-1                                 column=cf1:col-1, timestamp=1448448598740, value=val-99                                                        
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0280 seconds

hbase(main):078:0> scan 'development:staging-src1', { VERSIONS => 10, REVERSED => true }
ROW                                    COLUMN+CELL                                                                                                    
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
 row-1                                 column=cf1:col-1, timestamp=1448448598740, value=val-99                                                        
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
2 row(s) in 0.0370 seconds

Note: The reversed scan is only operating on the rows! Within the row the columns are still sorted ascending. This is caused by the internal data structures (the HFile mainly) being specifically built to sort ascending internally (and descending for versions). This cannot be easily reversed. We now will delete a column to show the internal ordering more:

hbase(main):079:0> delete 'development:staging-src1', 'row-1', 'cf1:col-2'
0 row(s) in 0.0110 seconds

A normal scan does not list the deleted cell:

hbase(main):080:0> scan 'development:staging-src1', { VERSIONS => 10 }
ROW                                    COLUMN+CELL                                                                                                    
 row-1                                 column=cf1:col-1, timestamp=1448448598740, value=val-99                                                        
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0230 seconds

But a special flag called “RAW” enables the retrieval of the stored delete marker and masked cell. This only works until the next compaction (one that does include removal of old data):

hbase(main):081:0> scan 'development:staging-src1', { VERSIONS => 10, RAW => true }
ROW                                    COLUMN+CELL                                                                                                    
 row-1                                 column=cf1:col-1, timestamp=1448448598740, value=val-99                                                        
 row-1                                 column=cf1:col-1, timestamp=1448447591017, value=val-1                                                          
 row-1                                 column=cf1:col-2, timestamp=1448449195466, type=DeleteColumn                                                    
 row-1                                 column=cf1:col-2, timestamp=1448447750441, value=val-2                                                          
 row-1                                 column=cf2:col-1, timestamp=1448447614823, value=val-1                                                          
 row-2                                 column=cf1:col-2, timestamp=1448447625100, value=val-2                                                          
2 row(s) in 0.0180 seconds

Cleaning up means disabling and dropping the tables, here the bulk versions:

hbase(main):082:0>  disable_all '.*'
development:staging-src1                                                                                                                               
development:staging-src1-clone                                                                                                                         
testtable                                                                                                                                              
testtable2                                                                                                                                             

Disable the above 4 tables (y/n)?
y
4 tables successfully disabled

hbase(main):083:0> drop_all '.*'
development:staging-src1                                                                                                                               
development:staging-src1-clone                                                                                                                         
testtable                                                                                                                                              
testtable2                                                                                                                                             

Drop the above 4 tables (y/n)?
y
4 tables successfully dropped

hbase(main):084:0> list
TABLE                                                                                                                                                  
0 row(s) in 0.0110 seconds

=> []

Next we create a namespace and table we need for the Hive example later on. It used Ruby again to generate test data:

hbase(main):085:0> create_namespace 'salesdw'
0 row(s) in 0.0330 seconds

hbase(main):086:0> create 'salesdw:itemdescs', { NAME => 'meta', VERSIONS => 5, COMPRESSION => 'Snappy', BLOCKSIZE => 8192 }, { NAME => 'data', COMPRESSION => 'GZ', BLOCKSIZE => 262144, BLOCKCACHE => 'false' }
0 row(s) in 0.4500 seconds

=> Hbase::Table - salesdw:itemdescs

hbase(main):087:0> require 'date'; import java.lang.Long
=> Java::JavaLang::Long
hbase(main):088:0> import org.apache.hadoop.hbase.util.Bytes
hbase(main):089:0>     def randomKey
hbase(main):090:1>       rowKey = Long.new(rand * 100000).to_s
hbase(main):091:1>       cdate = (Time.local(2011, 1,1) + rand * (Time.now.to_f - Time.local(2011, 1, 1).to_f)).to_i.to_s
hbase(main):092:1>       recId = (rand * 10).to_i.to_s
hbase(main):093:1>       rowKey + "|" + cdate + "|" + recId
hbase(main):094:1> end
hbase(main):095:0> 1000.times do
hbase(main):096:1*   put 'salesdw:itemdescs', randomKey, 'meta:title', ('a'..'z').to_a.shuffle[0,16].join
hbase(main):097:1> end
0 row(s) in 0.0210 seconds

0 row(s) in 0.0090 seconds

...

0 row(s) in 0.0040 seconds

0 row(s) in 0.0030 seconds

=> 1000

hbase(main):099:0> scan 'salesdw:itemdescs', LIMIT => 2
ROW                                    COLUMN+CELL                                                                                                    
 10000|1398671027|6                    column=meta:title, timestamp=1448449475573, value=rjhpsqzamtblcnfg                                              
 10006|1401686413|4                    column=meta:title, timestamp=1448449473176, value=fbvcygxlkzquteds                                              
2 row(s) in 0.0140 seconds

hbase(main):100:0> quit


Code

Use book example code:

$ cd ~
$ git clone https://github.com/larsgeorge/hbase-book.git
$ cd hbase-book/
$ mvn package install
$ ch03/bin/run.sh client.PutExample
$ ch03/bin/run.sh client.ScanExample
$ ch05/bin/run.sh admin.NamespaceExample

YCSB

Clone YCSB:

$ cd ~
$ git clone https://github.com/brianfrankcooper/YCSB.git

Unpack binary:

$ cd YCSB/
$ mvn package -DskipTests
$ cd ..
$ tar -zxvf YCSB/distribution/target/ycsb-0.6.0-SNAPSHOT.tar.gz 
$ cd ycsb-0.6.0-SNAPSHOT/

Test with basic workload:

$ seq -w 0 99 | sed 's/^/user/' > splits.txt
$ echo "create 'usertable', 'cf1', { SPLITS_FILE => 'splits.txt'}" | hbase shell
$ bin/ycsb load hbase10 -P workloads/workloada -p columnfamily=cf1 

Performance Evaluation (PE)

Run PE, loading data:

$ hbase pe
$ #hbase pe sequentialWrite 1
$ hbase pe --nomapred sequentialWrite 1
$ hbase pe --nomapred --compress=SNAPPY --valueZipf=1024 sequentialWrite 1

Check in shell:

$ hbase shell

Read data:

$ hbase pe --nomapred randomRead 1

Configuration Tuning

Block Cache:

<property>
   <name>hbase.bucketcache.combinedcache.enabled</name>
   <value>true</value>
 </property>
 <property>
   <name>hfile.block.cache.size</name>
   <value>0.2</value>
 </property>
 <property>
   <name>hbase.bucketcache.ioengine</name>
   <value>offheap</value>
 </property>
 <property>
   <name>hbase.bucketcache.size</name>
   <value>1024</value>
 </property>

Hive

[cloudera@quickstart hbase-book]$ hive
...
hive> create table pokes (foo int, bar string);
OK
Time taken: 2.512 seconds

hive> load data local inpath '/usr/share/doc/hive-1.1.0+cdh5.4.2+127/examples/files/kv1.txt' overwrite into table pokes;
Loading data to table default.pokes
Table default.pokes stats: [numFiles=1, numRows=0, totalSize=5812, rawDataSize=0]
OK
Time taken: 1.037 seconds

hive> create table hbase_table_1 (key int, value string) stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' with serdeproperties ("hbase.columns.mapping" = ":key,cf1:val", "hbase.table.name" = "hbase_table_1");
OK
Time taken: 1.859 seconds

hive> insert overwrite table hbase_table_1 select * from pokes;
Query ID = cloudera_20151124060505_ce4026c9-28d8-42c6-81e6-a33977dac2e1
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_1448350936237_0001, Tracking URL = http://quickstart.cloudera:8088/proxy/application_1448350936237_0001/
Kill Command = /usr/lib/hadoop/bin/hadoop job  -kill job_1448350936237_0001
Hadoop job information for Stage-0: number of mappers: 1; number of reducers: 0
2015-11-24 06:06:07,837 Stage-0 map = 0%,  reduce = 0%
2015-11-24 06:06:17,325 Stage-0 map = 100%,  reduce = 0%, Cumulative CPU 3.72 sec
MapReduce Total cumulative CPU time: 3 seconds 720 msec
Ended Job = job_1448350936237_0001
MapReduce Jobs Launched: 
Stage-Stage-0: Map: 1   Cumulative CPU: 3.72 sec   HDFS Read: 16065 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 3 seconds 720 msec
OK
Time taken: 26.291 seconds

hive> select * from hbase_table_1 limit 2;
OK
0    val_0
10    val_10
Time taken: 0.254 seconds, Fetched: 2 row(s)

hive> select count(*) from pokes;
...
OK
500
Time taken: 25.981 seconds, Fetched: 1 row(s)

hive> select count(*) from hbase_table_1;
...
OK
309
Time taken: 31.506 seconds, Fetched: 1 row(s)

hive> select count(distinct foo) from pokes;
OK
309
Time taken: 25.894 seconds, Fetched: 1 row(s)

hive>
