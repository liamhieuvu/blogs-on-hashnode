## Optimize AWS Cost in Practice

A dev-ops always has a task to optimize cloud server cost. This blog will show you how to manage and determine cost issues in AWS.

# Check monthly cost

First, go to https://console.aws.amazon.com/cost-management/home#/custom, we need to know the trend of cost. Is it increasing or unchanged? In my case, it is increasing.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657018575477/dBFNX7gX4.png align="left")

Next, we need to break down for more information. Usually, EC2, RDS and data transfer having the most significant cost. Select: group by service and bar chart. Service helps us breaking cost down. Bar chart helps us know the trend of each service type.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657019003183/fJke_M2p_.png align="left")

# Check detail cost

In previous break down cost image, EC2-Instance costs the most. It may be someone creating new EC2 or auto scaling feature. The second place is EC2-Other, which is volume disk, internal data transfer. The third and fourth places are RDS (database) and cache (redis).

We will break down EC2-Instance cost as below image. As you see, `USE2-DataTransfer-Out-Bytes` costs a lot. This is cost of data transfer from AWS to internet outside. We can also check instance cost. If an instance type has increasing cost, it may be auto scaling.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657019615433/BK66o8mci.png align="left")

Next, we break down EC2-Other cost as below image. The `VolumeUsage.*` types indicate disk for EC2 instances, and they seems normal. But, `DataTransfer-Regional-Bytes` cost is increasing over months. This is the cost of data transfer within region. In my case, blockchain dApps consume a lot of data from  self-hosted full nodes.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657020117129/EPalu_F1X.png align="left")

We can check how much data transfer from a node in `Monitoring` tab of a specific EC2 instance:


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657021487679/BAgji7Mwe.png align="left")

Next, we can break down RDS and ElastiCache cost as above to check abnormal cost.

# Actions

1. Due to medium business, I decided to turn off auto scaling of K8S cluster and adjust a suitable number of nodes. Remove used EC2 instance.

2. Remove gateway to reduce internal region transfer between blockchain full nodes and gateway. Applications will call directly to nodes.

3. All application should use internal address to call each others, don't call public DNS. If your case is to transfer images, use CloudFront (CDN) for caching. 

4. Reduce configuration of low traffic database.

5. Reduce configuration of Elastic cache. Use self-deployed redis on K8S for small applications.

You can read more at: https://aws.amazon.com/aws-cost-management/aws-cost-optimization/

Here is the result, remember to choose daily data. Don't wait to end of the month to check cost :))

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1657021575656/0c0_N-hgM.png align="left")
