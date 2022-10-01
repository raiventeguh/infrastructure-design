# infrastructure-design

# Assumption
1. Small Scale
2. Cost Conscious

# Frontend
Stack: 
- React or other framework should be fine
- Static Site 
- Cloudfront & S3 hosting

Frontend usually can be static site instead of Server or Incremental or Client Side
the difference is on how handling request. I probably won't divulge on each scenario in here. 

The Flow is
## Building Proces
- CI/CD Trigger
- Framework will generate the static site 
- Put the static site into S3 
- Invalidate Cloudfront (so cloudfront will re-cache)

# Backend
The idea of backend is to use Modular Monolith instead of Microservices
At smaller scenario, building microservices is realy costly
since we go with assumption on being cost conscious. 

## Network
Stack for Backend in network we will use
1. VPC: Multi-AZ (2AZ)
2. Subnet: Public
3. Cloudmap
4.  API gateway 
### How to Access API 
API gateway will be our gateway for this design. 
There is a limitation on API Gateway for request/s -> around (1k - 10k/s IRC)
By the assumption small company, 1k is a lot so API Gateway still viable option while it's still cheaper than Load Balancer. 

API Gateway send the request to Fargate(compute) by utilizing cloudmap. 

Cloudmap will handle the request (DNS) and have routing policy: 
- Simple, Multivalue, IP, Weighted, and other
we could use simple first while trying to see what kinda of traffic that we looking for. 

### How to Access Internal Server
Accessing Internal server is similar with Accesing API the differences is, Internal API don't need to be routed to API gateway again. Each Backend Service can access by hitting the Cloudmap (Internal DNS)
## Compute
Since we are building this on top of Modular Monolith, doing containeraized solution will bring benefit. We could do Kubernets or Docker, however depending the team and learning curve (assumming everyone is new) docker is easier to learn instead of Kubernets. 
Kube more powerful tho and easier to manage than docker

Specfic for compute: 
- Fargate with AutoScaling
- Small Task Compute & memory 
- Sidecar for tracing 
## Migrations
Postgresql will be reside in private subnet, to ensure security reason (can't be accessed by anyone over internet)
Due to this reason having a migration pipeline (Db Versioning) is better solution
there are few DB Migration for postgres, the major one is sqitch and Liquibas. (assume with liquibase) 

So the codepipeline will handle the orchestration, but the process will be handled in Codebuild. 
Codebuild: 
- private subnet
- amazonlinux
- small to medium -> probably medium 
Need to be in private subnet, to access db
Notes: Since codebuild in Private subnet, it will take more time to provisioning. roughly almost 100% longer (usually around 40 - 60second -> change to almost 120s for provisioning)


## Secret
Parameter store or Secret manager
- both can works
- parameter store is more static
- secret manager is more dynamic can be rotated

Note: rotate secret manager a bit tricky before, and can make some downtime (haven't check this again)
- due to this can make downtime for the time being parameter store for static DB or other creds is still preferable

## Monitoring and Logging
Logging will be using cloudwatch logs, Most of AWS service can integrate seamlessly with Cloudwatch.

Monitoring will be using X-ray for distributed tracing, Prometheus or Cloudwatch metrics(based on preference) and Grafana for Visualization 

This approach is vendor locked, and using grafana so we could share the dashboard to a lot of user instead of cloudwatch metrics. 

Notes: Cloudwatch Chart is kinda bad right now, so grafana is preferred (but adding extra cost for license every month)
