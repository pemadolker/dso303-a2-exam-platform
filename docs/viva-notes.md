# Viva notes

My own revision notes for the viva. Short answers I should be able to give without looking.

## What the assignment asked for
1. **Design** a production-grade AWS system for online exams: diagram, two request walkthroughs and a short explanation of the exam peak and AZ loss.
2. **Document** each service: what it does, why I picked it over another option, how it is configured and what it connects to.
3. **Build** it in the Cloud Architecture Lab and simulate both walkthroughs and one AZ failure. The code (Terraform, app, pipeline) is in the repo, but it did not have to be deployed on the Learner Lab.

## Ideas I must be able to explain
| Idea | My one-line version |
|---|---|
| Availability Zone | A separate data centre (or group of them) in the same region with its own power and network. Two AZs means one can fail and the other keeps going. |
| Multi-AZ RDS | A second copy of the database in the other AZ. Each write is copied before it is confirmed (synchronous), and AWS promotes the copy by itself if the primary dies. |
| Public vs private subnet | A public subnet has a route to the internet gateway, a private one does not. My data subnets have no route out at all. |
| NAT gateway | Lets private resources make outbound calls without being reachable from outside. One per AZ, so losing one AZ doesn't cut the other one off. |
| Presigned POST | A short-lived signed upload form from the API. The browser sends the file straight to S3, and S3 checks the size (max 50 MB) and the type itself. |
| SQS + DLQ | A queue holds the work. If a message fails 3 times it moves to the dead-letter queue and an alarm fires, so nothing gets lost quietly. |
| Idempotent | Doing it twice gives the same result as doing it once. Autosave and the workers are written like this because retries will happen. |
| RPO vs RTO | RPO is how much data we can afford to lose (5 min). RTO is how long we can be down (1 h). |
| Target tracking vs scheduled scaling | Target tracking reacts to load. Scheduled scaling gets ready before a known event. I use both because reacting alone is too slow for 3,000 students in 5 minutes. |
| Terraform state | Terraform's record of what it created. Kept in S3 so my laptop and the pipeline share the same one. |
| OAC (Origin Access Control) | Lets only my CloudFront distribution read the private frontend bucket. The bucket itself is not public. |
| X-Origin-Verify header | A secret header CloudFront adds. The ALB forwards only requests that carry it, so nobody can skip CloudFront and WAF by hitting the ALB directly. |

## Questions I expect
- **Why Fargate and not Lambda for the API?** Autosave is about 100 requests a second. With Lambda each concurrent copy opens its own database connection. Containers keep a small shared pool (5 per task), so the database sees a fixed number of connections.
- **Why not put the database in a public subnet?** Anything with a route to the internet can be attacked. Only the app and Lambda security groups can reach port 5432.
- **What happens if AZ A dies mid-exam?** The ALB health checks fail within about 20 s and traffic goes to the AZ B tasks. RDS promotes the standby in 1 to 2 minutes. The browser keeps the answers and retries the autosave. I showed this in the simulator with the AZ-A outage, and it still returned HTTP 200.
- **How do you stop a student changing the timer?** The deadline is saved on the server when the exam starts. The browser only displays it, and the API rejects answers after the deadline plus 30 s grace. A sweeper runs every 5 minutes and submits attempts whose time ran out.
- **How do you keep records for 5 years?** The S3 lifecycle moves files to cheaper storage and deletes them at 1,826 days. AWS Backup takes a monthly database copy kept for 5 years, because normal RDS backups only go up to 35 days.
- **Why did the simulator mark the ALB to API hop as HTTP?** TLS ends at the ALB. The last hop is inside the private subnet and only the ALB's security group can reach the tasks. That is a normal AWS pattern.
- **Why no RDS Proxy or ElastiCache, when the simulator suggested them?** The connection count is already capped in code (about 50 in the exam window, 80 at the maximum). Autosave is writes, and writes can't be cached. I'd add RDS Proxy if the API moved to Lambda.
- **Why ap-south-1?** It is the closest AWS region to Bhutan, so lower latency. CloudFront's certificate and its WAF still have to be in us-east-1, because that is an AWS rule for CloudFront.
- **What isn't perfect?** Capacity is calculated, not load tested. SES starts in the sandbox. Without a custom domain, CloudFront to ALB is HTTP inside AWS. The simulator can't model SES, so the last email step is drawn but not simulated. (Better to say this myself before being asked.)
- **Walk me through a push to main.** Tests and `terraform validate` run first. If deploy is switched on: Terraform applies, a new image tagged with the commit SHA is pushed to ECR, ECS swaps the tasks gradually and rolls back if the new ones are unhealthy, the frontend is synced to S3 and the CloudFront cache cleared, then a smoke test checks the site.

## Numbers worth remembering
- 3,000 students in 5 min = about 10 exam starts a second.
- Autosave: 3,000 students / 30 s = about 100 requests a second.
- 99.9% = about 43 minutes of downtime a month.
- API tasks: 2 normally, 6 in the exam window (3 per AZ), 12 at most.
- ALB health check: every 10 s, 2 failures = unhealthy, so about 20 s to stop sending traffic to a dead AZ.
- Lifecycle: Standard-IA at 30 days, Glacier at 365, delete at 1,826 (5 years).
