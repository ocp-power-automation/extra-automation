# PROMETHEUS QUERIES FOR MONITORING OF OC CLUSTER   

Prometheus provides a functional query language called PromQL (Prometheus Query Language) that lets the user select and aggregate time series data in real time. The result of an expression can either be shown as a graph, viewed as tabular data in Prometheus's expression browser, or consumed by external systems via the HTTP API.This document discusses about some Prometheus Queries which might help the users for monitoring and tracking their clusters.    


## 1. Overall I/O load on your instance



The below expression is used for the global health-check of our system, we will be able to know the overall I/O load on our system. The node exporter exposes the following metric :    
    `rate(node_disk_io_now[5s])`  
 and computing the rate for this metric will give the overall I/O load. Here, `[5s]` represents the time duration `d` of the query to be executed. Higher `d` smooths the graph, while lower `d` makes the graph more precise.   


***

## 2. Node disk's I/O Queries  


 Below are the various I/O queries made on the node disks:     
 
`node_disk_io_time_seconds_total` -- It serves as a counter and is incremented when I/O is in progress and hence is used to calculate disk 
 I/O utilisation.    
`node_disk_read_bytes_total` -- It gives the total bytes read by the I/Os.    
`node_disk_written_bytes_total` -- It shows the number of bytes written by the I/Os.       
`node_disk_read_time_seconds_total` -- It shows the total time taken by a read I/Os.     
`node_disk_write_time_seconds_total` -- This is the total number of seconds spent by all writes.        
`node_disk_reads_completed_total`  -- It shows the number of read I/Os that are completed.        
`node_disk_writes_completed_total` -- The total number of writes completed successfully.       
   

***

## 3. Comparing current data with historical data  


PromQL allows querying historical data and combining / comparing it to the current data. Just add offset to the query. For instance, the following query would return week-old data for all the time series with node_network_receive_bytes_total name:      
`node_network_receive_bytes_total offset 7d`  

***

## 4. etcd Monitoring   


etcd is an open source distributed key-value store used to hold and manage the critical information that distributed systems need to keep running. Most notably, it manages the configuration data, state data, and metadata for Kubernetes.     
If the etcd service does not run correctly, successful operation of the whole OpenShift Container Platform cluster is in danger. Therefore, it is reasonable to configure monitoring of etcd.      

Some of the metrics that are available and useful in our experience are as follows:    
`etcd_cluster_version `(gauge) : It shows which version is running.    
`etcd_debugging_disk_backend_commit_rebalance_duration_seconds` (cumulative) : The latency distributions of **`commit.rebalance`** called by bboltdb backend. (sum)    
`etcd_debugging_lease_revoked_total `(cumulative) : The total number of revoked leases.    
`etcd_debugging_lease_ttl_total `(cumulative) : Bucketed histogram of lease TTLs. (sum)    
`etcd_debugging_mvcc_db_total_size_in_bytes `(gauge) : Total size of the underlying database physically allocated in bytes.    
`etcd_debugging_mvcc_delete_total` (cumulative) : Total number of deletes seen by this member.    

***

## 5. Resource Metric: Kubernetes CPU Usage   


The below expression will give the CPU usage for all pods belonging to the cluster.      
`sum(rate(container_cpu_usage_seconds_total{container_name!="POD",pod_name!=""}))`   


***

## 6. Resource Metric: Kubernetes Container CPU usage  


The first expression will show CPU usage for individual instances of each container whereas the second expression aggregates CPU usage for all containers with the same name.      
a. `rate(container_cpu_usage_seconds_total{container_name!="POD",pod_name!=""})`    
b. `sum(rate(container_cpu_usage_seconds_total{container_name!="POD",pod_name!=""})) by (container_name)`     


