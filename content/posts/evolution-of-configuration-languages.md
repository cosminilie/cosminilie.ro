---
tags:
- DevOps
- Pulumi
- Configuration Management
- AWS CDK
title: "Is the golden age of configuration languages ending, and taking DevOps with it?"
date: 2020-11-18T10:12:14+02:00
draft: false
---

I think DevOps, as we understand it today, is coming to an end. At least the Ops part of it. As cloud infrastructure becomes a key application concern, more and more ops tasks are done by the cloud itself or built into the application. What's left is the provisioning and management of the infrastructure needed by the application. This comes with all the baggage associated, like security and networking, for example. 

All of these patterns that are now being forged will make the infrastructure provisioning indistinguishable from the application itself for the general use case. Right now, in most companies, traditional IT Groups have rebranded themselves as DevOps and are now the gatekeepers of AWS, instead of the on-premise VMWare clusters. They use Terraform instead of bash scripts and are generally more agile, having adopted many of the development practices. These are specialized people that are familiar with networking, with how IAM works in AWS. They are responsible for building the networking part, the infrastructure, securing it, and handing it over to the application that uses them. 

The developers are mostly fine with it, as they don't want to learn the intricacies of AWS IAM or VPC's. I'm reminded in many ways of how the database was treated when I joined the industry in the 2000s. Back then, the application would not touch the structure of the database. You would have scripts that would be run by DBAs on production systems. Those would create the database structure, the tables, indexes, and basically the entire database structure. Developers would then map these to their code and just perform DML once the application was running on a fixed schema that is managed by someone else.  I see infrastructure the same way today. 

Once ORM and various other frameworks started to manage this domain, these concerns got bundled in the application. There is no reason why the same thing won't happen to infrastructure. The only problem is we didn't have the right frameworks to do it, but now we're starting to get them.

The need to handle application infrastructure (I see this different than managing core business services infrastructure like AD, email, finance systems, etc.) at scale happened in the age of virtualization and started with CFEngine. For those that don't know, CFEngine was one of the first configuration management systems as we understand it today, followed by Puppet, Chef, and others. CFEngine happened in the age where most provisioned happened either by hand or by following documented step by step instructions. At the core of the proposition was a whitepaper written by Mark Burgess [^1]. The mainline of reasoning was:

> Present day computer systems are fragile and unreliable. Human beings are involved in the care and repair of computer systems at every stage in their operation. This level of human involvement will be impossible to maintain in the future. Biological and social systems of comparable and greater complexity have self-healing processes that are crucial to their survival. It will be necessary to mimic such systems if our future computer systems are to prosper in a complex and hostile environment.

CFEngine was one of the first cracks of the problem of building a self-healing system that works at scale and shares some of the ideas from the immune system. To achieve this, it leveraged four important properties:
 1) It used a DSL to describe the desired state instead of using procedural languages. This would try to abstract components to allow administrators to parameterize and reuse. 
 2) It has converging semantics, i.e., one describes what a system should look like, and when the system has been brought to that state, CFEngine becomes inert.
 4) It offers protection against an unfortunate repetition of tasks and hanging processes in situations where several administrators are working independently with little opportunity to communicate.

Using these features, one could achieve a self-correcting and self-healing system, which would result in a system that maintains fault tolerance. Today with AWS, we can design a system that displays the same properties by leveraging multi-regional services. By their nature, these services can transfer these properties to the application if employed with careful design. 


Lots of other tools appeared around the same time or soon after, each focusing on different aspects of the initial value proposition. Probably the most popular are Puppet and Chef. Each has its strong points. At the company I work, we went with Puppet to handle our infrastructure configuration mostly because it was easier to read for non-programers. From a sysadmin perspective, it was appealing to get something done without diving too much into coding. In time, it would prove to be a false choice that would hurt us more than it helped. 

Puppet has its own DSL, its own terminology, and idiosyncrasies. It has its own ecosystem of tools, dashboards, and extensions that help you get pretty far in managing your infrastructure. Its strong point is dealing with single server systems that didn't need distributed coordination between them. There is a way of coordination across several servers with exported resources and PuppetDB, but it always felt hacky to me((it may be a different story today, but I haven't followed the space for a couple of years now).

```puppet

#https://github.com/voxpupuli/puppet-minecraft/blob/master/manifests/user.pp

class minecraft::user {
  group { $minecraft::group:
    ensure => present,
    system => true,
  }

  user { $minecraft::user:
    ensure     => present,
    gid        => $minecraft::group,
    home       => $minecraft::install_dir,
    managehome => true,
    system     => true,
    require    => Group[$minecraft::group],
  }

  # Ensures deletion of install_dir does not break module, setup for plugins
  $dirs = [$minecraft::install_dir, "${minecraft::install_dir}/plugins"]

  file { $dirs:
    ensure  => directory,
    owner   => $minecraft::user,
    group   => $minecraft::group,
    require => User[$minecraft::user],
  }
}
```


This worked well and allowed us to build a lot of automation to manage hundreds of servers when dealing with simple infrastructure components. But like most things, it's the edge cases that make you feel the pain. Since the Puppet language is a DSL, simple problems started to become big problems. Like, not being able to do a proper for-loop. Or doing string manipulation. 

Most configuration languages suffer from these problems. In the end, you will also probably get in a situation where you need to extend them to cover a specific use case, and usually, it's at this point that you will need to write real code. There is only so much you can do without writing in "real" languages, so we should have probably done it from the start.  Looking back, probably Chef would have been a much better choice. At least with it, a lot of the things learned there could have been applied to other aspects of my career (Chef uses ruby instead of their own DSL). I don't have a lot of experience with Ansible or with Salt, but I feel they suffer from the same shortcomings and have similar strong points.

A somewhat distinct category in the configuration language space is Terraform and AWS Cloud Formation. They both borrow from the legacy of configuration management systems and try to sync their internal view of how things should look with reality. The main difference, outside of the way you express intent (which is still using DSL and not full fledge languages), is that they're designed to work at the cloud provider level. 

Just as Puppet and Chef are very good at managing typical resources on a machine (service, package, config file), Terraform and AWS Cloud Formation are very good at managing cloud services. They were built with the notion of cloud infrastructure.  This doesn't mean that you can't do the same thing with Puppet, Chef, and the rest of the gen2 configuration languages. You can, but due to how fast they adapted to the cloud reality, how they tried to monetize the cloud, and so on, Terraform became the uncontested king of DSL based cloud infrastructure management, while Cloud Formation reigned in AWS land.

All these tools adopted the best parts from CFEngine, the most important being the notion of convergent state. They would take your expressed intent, compare it to the machine, figure out any dependencies and order of steps that will bring the resources to its desired state. They usually also involve a compilation stage where they map the DSL to inner logic and create an execution plan. This would also catch basic mistakes.  These are good ideas that have battle-proven and are now the default way of dealing with infrastructure.


As we move forward, however, fundamental shifts are happening in the way we consume and deal with infrastructure components. Now you can leverage AWS services. You can build a very complex application that uses CloudFront for static content distribution, API gateway with Lambda to build API routes and add business functionality to them. Identity management can be handled by Cognito. On the backend, these Lambda functions can interact with an entire ecosystem of infrastructure directly, like RDS or DynamoDB. You can interact with analytics systems via Redshift or display visualized data via QuickSight. Background tasks can be handled by Lambda as well, workflows driven with step functions, async batch, and ETL tasks using AWS EMR or Glue. Composing these to achieve significant business value can be done with relatively little glue code as AWS services are very well integrated together.

New classes of applications are emerging that try to better adapt to this landscape. One example is what AWS labeled as serverless. Provisioning these types of apps using Terraform or Cloud Formation can lead to a lot of friction.  In such a tightly integrated model, your infrastructure will evolve with your application, so this probably needs to be one of the most important application concerns. AWS is pushing the SAM model here, but I can see a future where frameworks similar to Ruby-on-Rails or Django treat the AWS infrastructure similar to a database. So, how would having something like "rails migrate" do for the cloud, what it does for the data schema look like? Probably pretty nice, to a Rails Developer.  For better or worst, I think we're heading down this path.

So are we there? I don't think so. As the dust settles on some common abstractions that you can use across all clouds, we might get there. There are some attempts like "go-cloud" from the go world, which tries to build an abstraction across the most common denominator, but it's an uphill battle.

For now, my bet is on Pulumi and their automation api. So let's backtrack a little.  Pulumi is a framework (you can call it a configuration language framework) that allows you to write code in a mainstream language like javascript and typescript, python, go, c#. It still uses the same concepts as other configuration languages and, most of the support is actually built on top of Terraform. Where it really gets interesting is that since you're writing code, you're writing code. You can leverage all the packages you like and all the programming paradigms you enjoy together with your IDE and tooling build in the language ecosystem.

One of the first examples they show on their website is:

```typescript
// Create a serverless REST API
import * as awsx from "@pulumi/awsx";

// Serve a simple REST API at `GET /hello`.
let app = new awsx.apigateway.API("my-app", {
    routes: [{
        path: "/hello",
        method: "GET",
        eventHandler: async (event) => {
            return {
                statusCode: 200,
                body: JSON.stringify({ hello: "World!" }),
            };
        },
    }],
});

export let url = app.url;
```

What this does, is create an API Gateway in AWS with a "/hello" route and proxies that to an AWS Lambda function written in javascript. This lambda function simply returns the code 200 and an HTML body containing a JSON object `{hello: "World"!}`. 

Let's look at another example:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

export = async () => {
    const config = new pulumi.Config("aws");
    const providerOpts = { provider: new aws.Provider("prov", { region: <aws.Region>config.require("envRegion") }) };

    const vpc = awsx.ec2.Vpc.getDefault(providerOpts);

    // Create a security group to let traffic flow.
    const sg = new awsx.ec2.SecurityGroup("web-sg", { vpc }, providerOpts);

    const ipv4egress = sg.createEgressRule("ipv4-egress", {
        ports: new awsx.ec2.AllTraffic(),
        location: new awsx.ec2.AnyIPv4Location(),
    });
    const ipv6egress = sg.createEgressRule("ipv6-egress", {
        ports: new awsx.ec2.AllTraffic(),
        location: new awsx.ec2.AnyIPv6Location(),
    });

    // Creates an ALB associated with the default VPC for this region and listen on port 80.
    const alb = new awsx.elasticloadbalancingv2.ApplicationLoadBalancer("web-traffic",
        { vpc, external: true, securityGroups: [ sg ] }, providerOpts);
    const listener = alb.createListener("web-listener", { port: 80 });

    // For each subnet, and each subnet/zone, create a VM and a listener.
    const publicIps: pulumi.Output<string>[] = [];
    const subnets = await vpc.publicSubnets;
    for (let i = 0; i < subnets.length; i++) {
        const getAmiResult = await aws.getAmi({
            filters: [
                { name: "name", values: [ "ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-*" ] },
                { name: "virtualization-type", values: [ "hvm" ] },
            ],
            mostRecent: true,
            owners: [ "099720109477" ], // Canonical
        }, { ...providerOpts, async: true });

        const vm = new aws.ec2.Instance(`web-${i}`, {
            ami: getAmiResult.id,
            instanceType: "m5.large",
            subnetId: subnets[i].subnet.id,
            availabilityZone: subnets[i].subnet.availabilityZone,
            vpcSecurityGroupIds: [ sg.id ],
            userData: `#!/bin/bash
    echo "Hello World, from Server ${i+1}!" > index.html
    nohup python -m SimpleHTTPServer 80 &`,
        }, providerOpts);
        publicIps.push(vm.publicIp);

        alb.attachTarget("target-" + i, vm);
    }

    // Export the resulting URL so that it's easy to access.
    return { endpoint: listener.endpoint, publicIps: publicIps };
};

```



This is a more complicated example[^2], but there is a simpler version of using Fargate[^3] as the compute part if this feels too complicated to follow. 

So let's walk through it a bit to get a sense of what it does. After first building an internal Pulumi context to know which region to use in AWS, it gets the networking part of an AWS VPC configured. This is not something trivial as it involves considerate domain knowledge of AWS internal networking to do, even manually. Still, it only requires a line of code, and it builds something more than decent.

```typescript
const vpc = awsx.ec2.Vpc.getDefault(providerOpts);
```

After it's done, you will have a default VPC, with private and public subnets, setups up an internet gateway and route tables so that the designated public subnet has its default route (0.0.0.0) pointing at the internet gateway. When we create EC2 instances in the public subnets, they will be reachable from the internet and will have an outbound internet connection, while instances in the private subnet will be accessible only in the VPC and won't have internet access. This is the setup AWS recommends and is secured by default.


Next, it creates a security group (and AWS EC2 feature, which works like a firewall rule) to allow only web traffic via ipv6 and ipv4 to resources that have the security group attached to them. In this case, it will be a load balancer (Application Load Balancer is the AWS product name). 

```typescript
    // Create a security group to let traffic flow.
    const sg = new awsx.ec2.SecurityGroup("web-sg", { vpc }, providerOpts);

    const ipv4egress = sg.createEgressRule("ipv4-egress", {
        ports: new awsx.ec2.AllTraffic(),
        location: new awsx.ec2.AnyIPv4Location(),
    });
    const ipv6egress = sg.createEgressRule("ipv6-egress", {
        ports: new awsx.ec2.AllTraffic(),
        location: new awsx.ec2.AnyIPv6Location(),
    });

    // Creates an ALB associated with the default VPC for this region and listen on port 80.
    const alb = new awsx.elasticloadbalancingv2.ApplicationLoadBalancer("web-traffic",
        { vpc, external: true, securityGroups: [ sg ] }, providerOpts);
    const listener = alb.createListener("web-listener", { port: 80 });


```

Now that we have our load balancer, we're going to wait for the subnets to be created (see the await keyword). Once this is done, we can loop through all public subnets, and in each subnet, create an EC2 instance using the ubuntu AMI.  For test purposes, using the `userData` script, we'll inject a small bash script that creates an HTML page. This starts a python embedded web server to serve it. Here we could have done anything (e.g. fetched a spring boot app from s3 and started, or any type of application, and run it). In the end, we would attach the EC2 instance to the ELB, and we're done.

```typescript
  const subnets = await vpc.publicSubnets;
    for (let i = 0; i < subnets.length; i++) {
        const getAmiResult = await aws.getAmi({
            filters: [
                { name: "name", values: [ "ubuntu/images/hvm-ssd/ubuntu-trusty-14.04-amd64-server-*" ] },
                { name: "virtualization-type", values: [ "hvm" ] },
            ],
            mostRecent: true,
            owners: [ "099720109477" ], // Canonical
        }, { ...providerOpts, async: true });

        const vm = new aws.ec2.Instance(`web-${i}`, {
            ami: getAmiResult.id,
            instanceType: "m5.large",
            subnetId: subnets[i].subnet.id,
            availabilityZone: subnets[i].subnet.availabilityZone,
            vpcSecurityGroupIds: [ sg.id ],
            userData: `#!/bin/bash
    echo "Hello World, from Server ${i+1}!" > index.html
    nohup python -m SimpleHTTPServer 80 &`,
        }, providerOpts);

        alb.attachTarget("target-" + i, vm);
    }


```

To resume, in this example, we created an AWS VPC with all networking configured according to AWS best practices. We've set up a load balancer, secured it to not allow non desired traffic, deployed a couple of AWS EC2 instances in each AWS availability zone to get fault tolerance (again an AWS best practice), and deployed our webpage. Neet right? We also did this without a "DevOps" engineer. Of course, like any domain-specific framework, some knowledge of the domain is required, but once you've studied a bit the SDK, the cloud is no different than any other framework that you're using.


Now, all this is nice, but how do you fit this into your own app? The answer, unfortunately, is that it's still being worked out.  But given that it will probably be in the same language as your application, it makes sense to keep everything in the same repo. It still requires a separate tool to run (Pulumi), but you can think of this like just another tool in the toolchain. If that is the case, other than using the same language for building my app and the cloud infrastructure that it uses, what's the point?  If I have to use a separate tool just for this, then it's not all that different than using Terraform, for example. This is where the Pulumi automation api comes into place. 


```typescript 
import { InlineProgramArgs, LocalWorkspace } from "@pulumi/pulumi/x/automation";
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as pulumi from "@pulumi/pulumi";
import * as mysql from "mysql";

const process = require('process');

const args = process.argv.slice(2);
let destroy = false;
if (args.length > 0 && args[0]) {
    destroy = args[0] === "destroy";
}



const run = async () => {
    // This is our pulumi program in "inline function" form
    const pulumiProgram = async () => {
        const vpc = awsx.ec2.Vpc.getDefault();
        const subnetGroup = new aws.rds.SubnetGroup("dbsubnet", {
            subnetIds: vpc.publicSubnetIds,
        });

        // make a public SG for our cluster for the migration
        const securityGroup = new awsx.ec2.SecurityGroup("publicGroup", {
            egress: [
                {
                    protocol: "-1",
                    fromPort: 0,
                    toPort: 0,
                    cidrBlocks: ["0.0.0.0/0"],
                }
            ],
            ingress: [
                {
                    protocol: "-1",
                    fromPort: 0,
                    toPort: 0,
                    cidrBlocks: ["0.0.0.0/0"],
                }
            ]
        });

        // example only, you should change this
        const dbName = "hellosql";
        const dbUser = "hellosql";
        const dbPass = "hellosql";

        // provision our db
        const cluster = new aws.rds.Cluster("db", {
            engine: "aurora-mysql",
            engineVersion: "5.7.mysql_aurora.2.03.2",
            databaseName: dbUser,
            masterUsername: dbName,
            masterPassword: dbPass,
            skipFinalSnapshot: true,
            dbSubnetGroupName: subnetGroup.name,
            vpcSecurityGroupIds: [securityGroup.id],
        });

        const clusterInstance = new aws.rds.ClusterInstance("dbInstance", {
            clusterIdentifier: cluster.clusterIdentifier,
            instanceClass: "db.t3.small",
            engine: "aurora-mysql",
            engineVersion: "5.7.mysql_aurora.2.03.2",
            publiclyAccessible: true,
            dbSubnetGroupName: subnetGroup.name,
        });

        return {
            host: pulumi.interpolate`${cluster.endpoint}`,
            dbName,
            dbUser,
            dbPass
        };
    };

    // Create our stack 
    const args: InlineProgramArgs = {
        stackName: "dev",
        projectName: "databaseMigration",
        program: pulumiProgram
    };

    // create (or select if one already exists) a stack that uses our inline program
    const stack = await LocalWorkspace.createOrSelectStack(args);

    console.info("successfully initialized stack");
    console.info("installing plugins...");
    await stack.workspace.installPlugin("aws", "v3.6.1");
    console.info("plugins installed");
    console.info("setting up config");
    await stack.setConfig("aws:region", { value: "us-west-2" });
    console.info("config set");
    console.info("refreshing stack...");
    await stack.refresh({ onOutput: console.info });
    console.info("refresh complete");

    if (destroy) {
        console.info("destroying stack...");
        await stack.destroy({ onOutput: console.info });
        console.info("stack destroy complete");
        process.exit(0);
    }

    console.info("updating stack...");
    const upRes = await stack.up({ onOutput: console.info });
    console.log(`update summary: \n${JSON.stringify(upRes.summary.resourceChanges, null, 4)}`);
    console.log(`db host url: ${upRes.outputs.host.value}`);
    console.info("configuring db...");

    // establish mysql client
    const connection = mysql.createConnection({
        host: upRes.outputs.host.value,
        user: upRes.outputs.dbUser.value,
        password: upRes.outputs.dbPass.value,
        database: upRes.outputs.dbName.value
    });

    connection.connect();

    console.log("creating table...")

    // make sure the table exists
    connection.query(`
    CREATE TABLE IF NOT EXISTS hello_pulumi(
        id int(9) NOT NULL,
        color varchar(14) NOT NULL,
        PRIMARY KEY(id)
    );
    `, function (error, results, fields) {
        if (error) throw error;
        console.log("table created!")
        console.log('Result: ', JSON.stringify(results));
        console.log("seeding initial data...")
    });
    
    // seed the table with some data to start
    connection.query(`
    INSERT IGNORE INTO hello_pulumi (id, color)
    VALUES
        (1, 'Purple'),
        (2, 'Violet'),
        (3, 'Plum');
    `, function (error, results, fields) {
        if (error) throw error;
        console.log("rows inserted!")
        console.log('Result: ', JSON.stringify(results));
        console.log("querying to veryify data...")
    });

    
    // read the data back
    connection.query(`SELECT COUNT(*) FROM hello_pulumi;`, function (error, results, fields) {
        if (error) throw error;
        console.log('Result: ', JSON.stringify(results));
        console.log("database, tables, and rows successfuly configured!")
    });

    connection.end();
};

run().catch(err => console.log(err));
```

This is a big example, in terms of lines of code longer, but the main point is: we create the infrastructure and table structure in the same code base. So let's walk through this a bit.


The first part takes care of the networking setup in AWS and creates a security group to allow all access.

```typescript
        const vpc = awsx.ec2.Vpc.getDefault();
        const subnetGroup = new aws.rds.SubnetGroup("dbsubnet", {
            subnetIds: vpc.publicSubnetIds,
        });

        // make a public SG for our cluster for the migration
        const securityGroup = new awsx.ec2.SecurityGroup("publicGroup", {
            egress: [
                {
                    protocol: "-1",
                    fromPort: 0,
                    toPort: 0,
                    cidrBlocks: ["0.0.0.0/0"],
                }
            ],
            ingress: [
                {
                    protocol: "-1",
                    fromPort: 0,
                    toPort: 0,
                    cidrBlocks: ["0.0.0.0/0"],
                }
            ]
        });


```

The second part of the infrastructure section creates the database instance using Aurora

```typescript

  // example only, you should change this
        const dbName = "hellosql";
        const dbUser = "hellosql";
        const dbPass = "hellosql";

        // provision our db
        const cluster = new aws.rds.Cluster("db", {
            engine: "aurora-mysql",
            engineVersion: "5.7.mysql_aurora.2.03.2",
            databaseName: dbUser,
            masterUsername: dbName,
            masterPassword: dbPass,
            skipFinalSnapshot: true,
            dbSubnetGroupName: subnetGroup.name,
            vpcSecurityGroupIds: [securityGroup.id],
        });

        const clusterInstance = new aws.rds.ClusterInstance("dbInstance", {
            clusterIdentifier: cluster.clusterIdentifier,
            instanceClass: "db.t3.small",
            engine: "aurora-mysql",
            engineVersion: "5.7.mysql_aurora.2.03.2",
            publiclyAccessible: true,
            dbSubnetGroupName: subnetGroup.name,
        });

        return {
            host: pulumi.interpolate`${cluster.endpoint}`,
            dbName,
            dbUser,
            dbPass
        };
    };

```
Now all of this is wrapped in a Pulumi "stack" and run to create the infrastructure:

```typescript
// Create our stack 
    const args: InlineProgramArgs = {
        stackName: "dev",
        projectName: "databaseMigration",
        program: pulumiProgram
    };

    // create (or select if one already exists) a stack that uses our inline program
    const stack = await LocalWorkspace.createOrSelectStack(args);

    //load pulumi plugins
    await stack.workspace.installPlugin("aws", "v3.6.1");
   
   //set the aws region and logging
    await stack.setConfig("aws:region", { value: "us-west-2" });
    await stack.refresh({ onOutput: console.info });

   //in case we run the program with the destroy flag to remove everythig
    if (destroy) {
        console.info("destroying stack...");
        await stack.destroy({ onOutput: console.info });
        console.info("stack destroy complete");
        process.exit(0);
    }

    //wait for the infra to be provisioned
    const upRes = await stack.up({ onOutput: console.info });


```

Once we have our infra up, we can start using it:

```typescript
 // establish mysql client
    const connection = mysql.createConnection({
        host: upRes.outputs.host.value,
        user: upRes.outputs.dbUser.value,
        password: upRes.outputs.dbPass.value,
        database: upRes.outputs.dbName.value
    });

    connection.connect();


    // make sure the table exists
    connection.query(`
    CREATE TABLE IF NOT EXISTS hello_pulumi(
        id int(9) NOT NULL,
        color varchar(14) NOT NULL,
        PRIMARY KEY(id)
    );
    `, function (error, results, fields) {
        if (error) throw error;
        console.log("table created!")
        console.log('Result: ', JSON.stringify(results));
        console.log("seeding initial data...")
    });
    
    // seed the table with some data to start
    connection.query(`
    INSERT IGNORE INTO hello_pulumi (id, color)
    VALUES
        (1, 'Purple'),
        (2, 'Violet'),
        (3, 'Plum');
    `, function (error, results, fields) {
        if (error) throw error;
        console.log("rows inserted!")
        console.log('Result: ', JSON.stringify(results));
        console.log("querying to veryify data...")
    });

    
    // read the data back
    connection.query(`SELECT COUNT(*) FROM hello_pulumi;`, function (error, results, fields) {
        if (error) throw error;
        console.log('Result: ', JSON.stringify(results));
        console.log("database, tables, and rows successfuly configured!")
    });

```


Where this model really shines is when creating "serverless" applications. Here the developers have created really nice abstractions, which make the workflow work seamless and shows us the way forward:

```typescript

// Define a new GET endpoint backed by a lambda function and a static folder mapped to / which will be saved in s3.
const api = new awsx.apigateway.API("test", {
    routes: [{
          path: "/",
          localPath: "www",
        },
        {
          path: "/test",
          method: "GET",
          eventHandler: async (event) => {
            // This code runs in an AWS Lambda anytime `/test` is hit.
            return {
                statusCode: 200,
                body: "Hello, API Gateway!",
            };
          },
    }],
})
```

What this does, is create an API gateway with 2 routes, one for root(`/`) and one for the `/test` endpoint. The `/` route will serve content from an s3 bucket, which will be uploaded from the local www directory when this program is run. The `/test` endpoint will be backed by a lambda function in which content is taken from the event handler code block. This is a trivial example, but given what we've discussed until now, it's very appealing.

While I've based most of my examples on Pulumi until now, they are not the only ones moving in this direction. For example, AWS has introduced the "AWS CDK"[^4] or cloud development kit. This allows you to write code in your language of choice and it will get "synthesized" at runtime into cloud formation stacks. There is even a "Construct Library" which allows you to use components already created by AWS and include them in your codebase. These components abstract a lot of the complexities (for example networking) into simple to use bits that are secured and follow best practices. 

```typescript
import * as core from "@aws-cdk/core";
import * as apigateway from "@aws-cdk/aws-apigateway";
import * as lambda from "@aws-cdk/aws-lambda";
import * as s3 from "@aws-cdk/aws-s3";

export class WidgetService extends core.Construct {
  constructor(scope: core.Construct, id: string) {
    super(scope, id);

    const bucket = new s3.Bucket(this, "WidgetStore");

    const handler = new lambda.Function(this, "WidgetHandler", {
      runtime: lambda.Runtime.NODEJS_10_X, // So we can use async in widget.js
      code: lambda.Code.asset("resources"),
      handler: "widgets.main",
      environment: {
        BUCKET: bucket.bucketName
      }
    });

    bucket.grantReadWrite(handler); // was: handler.role);

    const api = new apigateway.RestApi(this, "widgets-api", {
      restApiName: "Widget Service",
      description: "This service serves widgets."
    });

    const getWidgetsIntegration = new apigateway.LambdaIntegration(handler, {
      requestTemplates: { "application/json": '{ "statusCode": "200" }' }
    });

    api.root.addMethod("GET", getWidgetsIntegration); // GET /
  }
}
```

This is how an AWS CDK sample does a similar thing to what we've done above with Pulumi,  look like. 

Even Terraform is moving in this direction with a project based on AWS CDK where you can write in typescript and python. These constructs, which underneath use Terraform modules, work to provision infrastructure across multiple cloud providers.


For better or worse, I think we're heading in a direction, where at best, the infrastructure will live with your code, similar to how build files live with it. But, I think people will try to push this further and integrate the infrastructure code inside the actual app. All of it will be managed, by the application itself, at runtime. Now there is a spectrum here, and like most things, different applications will be somewhere along amidst that. The application type will dictate most of this. For example, I'm finding it difficult to imagine this will make as big of an impact on monolithic java apps that are backed by Postgres instance as it would make on a serverless app that runs in AWS.

What this means for DevOps engineers right now will also vary. Similar to how some companies understand to create a DevOps engineer role which is in charge of building ci pipelines, there will be companies where developers will take over the entire responsibility and use the tools and practices mentioned here to do it all. As someone involved in DevOps for most of my career, my advice is to learn typescript and get familiar with Pulumi. It will take time to get here, but the writing is on the wall. At least, if this doesn't play out you, can always switch to a frontend developer as I'm sure they need for them will grow and grow. AWS sucks at UX and won't come after your job.

[^1]: [Computer Immunology - Mark Burgess](https://www.usenix.org/legacy/event/lisa98/full_papers/burgess/burgess.pdf)

[^2]: https://github.com/pulumi/pulumi-awsx/tree/master/nodejs/examples/alb/ec2Instance

[^3]: https://github.com/pulumi/pulumi-awsx/tree/master/nodejs/examples/alb/fargate

[^4]: https://aws.amazon.com/cdk/
