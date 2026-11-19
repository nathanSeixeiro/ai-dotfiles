# AWS accounts, profiles and network map

Read from `~/.aws/config` (profile names, account ids, regions only) and `k8s-infra` docs/steering, 2026-11. Never read `~/.aws/credentials` or `~/.aws/sso/cache`.

## Login

SSO session `sso` (start URL `https://d-90662173a0.awsapps.com/start`, sso_region us-east-1). All org profiles use role `AdministratorAccess`.
- Login: `aws sso login --sso-session sso` (one login covers every profile).
- Check: `aws sts get-caller-identity --profile <profile>`. Expired token = ask the user to run the login (it opens a browser).
- The role is admin. Read-only is enforced by THIS agent's rules, not by IAM. Treat every non-read command as forbidden.

## Focus profiles (default region us-east-1)

| Env | Profile | Account | What lives there |
|---|---|---|---|
| DEV | `development-us` | 111111111111 | EKS `eks-dev`, vpc-eks-dev, ECR dev, app S3/IAM/ElastiCache (aws-infra `accounts/dev`) |
| HML (+ QA) | `homologation-us` | 222222222222 | EKS `eks-hml`, vpc-eks-hml, ECR hml (QA images suffix `-qa`), aws-infra `accounts/hml` |
| PRD | `production-us` | 333333333333 | EKS `eks-prod`, ECR prd, aws-infra `accounts/prd`. Most careful account |
| Network | `network-us` | 444444444444 | Transit Gateway, CloudFront + WAF, ALBs (HML internet-facing, DEV internal), VPC Endpoint Services (PrivateLink), Route 53 Resolver rules shared via RAM |

Always pass `--profile` explicitly. Never rely on `default`. Region: `--region us-east-1` unless the user names São Paulo.

## Other profiles (exist, out of focus: only when the user asks, read-only)

- `*-sp` twins of every account (region sa-east-1): `development-sp`, `homologation-sp`, `production-sp`, `network-sp`.
- `backup-us/sp` 555555555555, `log-us/sp` 999999999999, `security-us/sp` 888888888888, `gsti-us/sp` 666666666666, `mycorp-us/sp` 777777777777.
- `eks-iam`, `zabbix-user`: non-SSO (static keys). Do not use. Mention if a task seems to need them.

## Network map (EKS)

```
Network 444444444444: CloudFront+WAF -> ALB (HML internet-facing / DEV internal) -> VPC Endpoint Service (PrivateLink)
                      Transit Gateway tgw-0774501afa81b25fe
                             |                     |                     |
vpc-eks-dev (DEV)        vpc-eks-hml (HML)       vpc-eks-prd (PRD, planned)
vpc-01bbd88037a2ca0aa    vpc-04f1323b385edae9f
172.20.40.0/22           172.20.44.0/22           primary: to define
pods 100.64.0.0/17       pods 100.64.128.0/17     pods 100.65.0.0/16
```

- Subnets per VPC: node (/24 x2, us-east-1b/1c), endpoint (/27 x2), tgw (/28 x2), pods (secondary CIDR /18 x2, VPC-CNI custom networking via ENIConfig per AZ).
- Route tables: `rtb-eks-<env>-private` 0.0.0.0/0 -> TGW (node, endpoint, pod subnets); `rtb-tgw-<env>` local only.
- 9 VPC endpoints per VPC: s3 (gateway), ec2, sts, ecr.api, ecr.dkr, eks (SG `vpc-endpoint-eks-sg`), ssm, ssmmessages, ec2messages (SG `ssm-endpoint-sg`). 443 from VPC CIDR.
- DNS: 14 Resolver forward rules shared via RAM to the VPCs: corp.local, corp.local, corp.example, dsp.local, desmi.sp, hdesmi.sp, corp.local, corp.local, corp.example, corp.example, corp.local, corp.local, app.corp.local. On-prem `.corp.local` DB names resolve through these. DNS issues: check rule association to the VPC first.
- EKS clusters: private endpoint only, service CIDR 10.100.0.0/16, auth `API_AND_CONFIG_MAP`, access entries via Terraform, Pod Identity preferred over IRSA. PRD node groups: linux t3a.xlarge, windows m5.large, `node-app-c` t3a.xlarge (label `NodeApp=app-c`). Karpenter module exists in infra.

Source of truth for this map: `k8s-infra/Eks/docs/resumo-terraform-eks-clusters.md` and `.kiro/steering/aws-networking.md`. If live AWS differs, live wins: report the drift.

## Who owns changes

| Resource | Repo (Terraform, CI apply) | Owner agent |
|---|---|---|
| EKS clusters, node groups, Karpenter, addons, EKS IAM | `k8s-infra/Eks` | terraform-specialist |
| App S3, IAM users/policies, ElastiCache, SGs | `aws-infra/accounts/<env>/<app>.tf` | terraform-specialist |
| K8s manifests on EKS | `k8s-gitops/Eks` | devops-agent |
| Network account (TGW, CloudFront, WAF, ALB, Resolver) | not found in local repos: ask the user who owns it | user |

You answer and recommend. Changes go to the owner agent as a PR.
