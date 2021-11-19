---
layout: post
title:  "Internal APIs with Fargate"
tags: ["aws", "infra", "nlb", "elb", "networking"]
header: "/img/SavingsPlansPricing.png"
comments: true
---

# A walk through on building an internal API using Fargate 

Recently, we needed to design and implement an API that supplied data to a Graphql resolver in our customer api ... the catch, the API was not in the same account.

There are many ways to solve this problem and this is just our take on it.

We were not keen on exposing the internal API publicly for a few reasons which mostly boil down to cost.

Firstly, having a public API means that anybody can hit the endpoint without getting any further due to authentication, and because anybody can hit it, it also means anybody can abuse it or DDOS it. We could have throws an AWS WAF at the problem but WAF will incur a cost and scaling out the solution across many accounts with many API's seemed like an expensive approach to the problem.

Secondly, we wanted to avoid doubling NATing costs between the customer api and the internal data api. As we know when data leaves AWS is where hidden costs live.
We know we are going to be transporting the data out of AWS over the customer api but we didn't want to get double hit with the cost of the data being egressed out fo the internal api also.

Firstly, I'll present the diagram

![Internal API Arch Diagram](/img/arch/api/internal-api.png)

The idea is quite simple. Using CDK and in particular the [NetworkLoadBalancedFargateService](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-ecs-patterns.NetworkLoadBalancedFargateService.html) construct from the ecs-patterns module does almost everything you need.  


First create a subdomain (api.acme.com) zone.
{% highlight typescript %}
{% raw %}
const hostedZone = new route53.PublicHostedZone(this, 'ConfigHostedZone', {
    zoneName: 'api.acme.com'
});
{% endraw %}
{% endhighlight %}

Next create the NetworkLoadBalancer that will front the Fargate Service.
{% highlight typescript %}
{% raw %}
let nlb = new NetworkLoadBalancer(this, "NetworkFargateServiceLB", {
            vpc,
            internetFacing: false,
            deletionProtection: false,
            crossZoneEnabled: true,
            loadBalancerName: "API-NLB",
            vpcSubnets: {
                subnetFilters: props.subnetFilters
            }
        });
{% endraw %}
{% endhighlight %}


Supply the subnet filters allowing only selected private subnets to be used.
I'll discuss the reason to supply our own NetworkLoadBalancer below.

Next create the Fargate service, specifying the image options and the NetworkLoadbalancer you created in the previous step.

{% highlight typescript %}
{% raw %}
        let nlbService = new ecs_patterns.NetworkLoadBalancedFargateService(this, "NetworkFargateService", {
            cluster: cluster, // Required
            cpu: 512, // Default is 256
            desiredCount: 2, // Default is 1
            taskImageOptions: {
                image: ecs.ContainerImage.fromEcrRepository(repository, props.imageTag),
                taskRole: role
            },
            memoryLimitMiB: 1024, // Default is 512
            publicLoadBalancer: false, // Default is false
            loadBalancer: nlb,
            taskSubnets: {
                subnetFilters: props.subnetFilters
            },
            domainName: props.hostedZoneName,
            domainZone: hostedZone,
            recordType: NetworkLoadBalancedServiceRecordType.ALIAS
        });

{% endraw %}
{% endhighlight %}

Passing the domainName and domainZone to the construct allows the construct to create the Alias record pointing to the NetworkLoadBalancer you supplied.

NetworkLoadbalancer exists only in a private subnet and is not exposed publicly it only has a private IP address which is what the DNS Alias will resolve to.

Finally you need to add an NS record to your domain name (acme.com) to delegate to your subdomain (api.acme.com).

When the customer api now makes a request to api.acme.com, the DNS lookup first reaches acme.com which then delegates to api.acme.com and then resolves to the private IP address of the NetworkLoadBalancer.



Great, but thats a private IP address. So how to get the private IP address to reach the Fargate Service.

This is where we make use of our Transit Gateway Attachments.

Our accounts include a VPC which is attached to a shared Transit Gateway.

After the customer api resolves the api.acme.com domain it will make a request to the private IP Address.
This request is directed by the route table on the VPC to the transit gateway which then routes the request to the correct attachment on the VPC in the account supplying the internal api.

This then routes to the NetworkLoadBalancer which in turn load balances on the targates supplied by the Fargate Service.

## <span style="color:#00008b;">Voilla!!! :)</span>



### <span style="color:#780a27; text-align: center">All seems simple but there are a few gochas.</span>

* For Fargate to reach ECR to pull down its docker image you need to include a VPC endpoint and this needs to be included in your route table. Assuming you are using Fargate 1.4.0 you will require two entries to your route table. You can read more [here](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)

* The CDK pattern out of the box it doesn't work using the NetworkLoadBalancer. You will notice your Fargate Tasks restarting and this is because there is a missing entry from the security group on the Fargate service. Don't worry there is an open issue [here](https://github.com/aws/aws-cdk/issues/1490). So if I have some time I may look into resolving. It can of course be fixed by adding the entry in your CDK.

{% highlight typescript %}
{% raw %}
nlbService.service.connections.allowFromAnyIpv4(Port.tcp(80), "Allowing HTTP connections through");
{% endraw %}
{% endhighlight %}

* For the eagle eyed observer might have noticed that the loadBalancer can be created by the ecs-pattern construct. Well it's not ideal in how it places the NetworkLoadBalancer. A NetworkLoadBalancer can only be deployed into *one subnet per AZ*. The construct will attempt to deploy to all subnets it finds. So providing the parameter publicLoadBalancer as false instructs the construct to deploy to all private subnets in the VPC. Maybe that seems reasonable, but your deployment will fall over if you have more than one private subnet in a given AZ. This is the reason we've supplied our own NetworkLoadBalancer to control the subnets it deploys to.

* Again with CDK, The last quirk I suppose we noticed was with the use of TLS. If you want to make the connection between your customer api and the internal API secure you need listen on port 443 on the NetworkLoadBalancer with a TLS cert from ACM. However the CDK ecs-pattern doesn't allow passing of the TLS certificates. As a result you are forced to add a second listener to the NetworkLoadBalancer. It's not the end of the world, but you now have port 80 opened on you loadbalancer without really needing it open.

{% highlight typescript %}
{% raw %}
let tlsCert = new acm.Certificate(this, "APIDomainCert", {
    domainName: props.hostedZoneName,
    validation: acm.CertificateValidation.fromDns(hostedZone)
});

let listener = nlb.addListener("HTTPS-Listener-NLB", {
    port: 443,
    protocol: Protocol.TLS,
    certificates: [{certificateArn: tlsCert.certificateArn}],

});

listener.addTargets("Fargate-Service-Target", {
    port: 80,
    protocol: Protocol.TCP,
    targets: [nlbService.service]
});
{% endraw %}
{% endhighlight %}
