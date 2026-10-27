**Resource decommission
**

EC2 instances
Unused EC2 instances will be discovered based on a age and low network traffic. aws-ec2-low-traffic plugin will be used. 

If any unused EC2 instance was discovered manually - same process applies.

Decomission flow:

Identify EC2 instances which potentially can be decomissioned

Contact owner of those instances to confirm. If owner cant be discovered based on tags SRE will create Jira case and assign it to @Alexey Kuksin. Alexey will help to identify owner or provide next steps for instance.

Instances which are under investigation by Alexey, should not be modified (suspended or deleted)

Add new tag investigation = JIRA-ID (e.g. investigation = MIST-123)

If instance CAN NOT be suspended (based on conversation with owner):

Add new tag comment= minigfull comment (e.g. comment = owner contancted, dont suspend, 30/08/2024)

Add new tag DO_NOT_SUSPEND = 1

Suspend instance for 3 weeks.

Add new tag suspended= DD/MM/YYYY (e.g. suspended = 18/07/2024)

Add new tag suspended_by= juniper_id (e.g. suspended_by = bkupidura)

Add new tag comment= minigfull comment (e.g. comment = no owner, delete after 01/08/2024, comment = owner contacted, delete after 01/08/2024, comment = owner contacted, dont delete

If instance can be suspended but not deleted (based on conversation with owner), add tag DO_NOT_DELETE= 1

Delete instance

Deletion can be executed only if ALL conditions are true

Instance have tag suspended with proper date of suspension

Instance have tag suspended_by with proper juniper id

Instance have tag comment with information that owner was contacted or owner is unknown 

Instance was suspended at least 3 weeks ago

Instance does not have DO_NOT_DELETE tag

Instance should be deleted with tool used for deployment (if possible). If instance is managed by Terraform it should be also removed by it.

Instances managed by Terraform will leave EBS volumes behind. Those volumes needs to be deleted manually.

Auto-Scaling-Group
Oversized ASGs will be identified based on low memory and cpu usage across all instances in group. THOSE RESOURCES WILL NOT BE REMOVED - THEY CAN BE ONLY SCALED DOWN. aws-asg-low-cpu-memory-usage plugin will be used to identify oversized ASGs.

ASG can be scaled down only by ONE flavor (m7a.2xlarge → m7a.xlarge) at a time.

Scale down flow:

Identify which ASG can be potentially scaled down

Identify which services are running in EC2 instances from this ASG

Check cpu and memory limits for all services. Sum of all limits should be lower than new flavor.

If resource limits are bigger than 85% of new flavor capacity, service limits should be decreased - this activity should be performed with service owners.

 Each ASG scaling activity should have dedicated Jira case. Jira should be approved by @David Kochiashvili, @Alexey Kuksin or @Rameshbabu Dandamudi

EBS volumes
Unused EBS volumes will be identified based on attachment status. If volume is reported as available it will be removed. aws-ebs-not-attachedplugin will be used

Decomission flow:

Identify EBS volumes which available state

Contact owner of those volumes to confirm. If owner cant be discovered based on tags SRE will NOT TRY to find owner.

Add below tags

Add new tag inspected= DD/MM/YYYY (e.g. inspected = 18/07/2024)

Add new tag inspected_by= juniper_id (e.g. suspended_by = bkupidura)

Add new tag comment= minigfull comment (e.g. comment = no owner, delete after 01/08/2024, comment = owner contacted, delete after 01/08/2024, comment = owner contacted, dont delete

If EBS can not be deleted (based on conversation with owner), add tag DO_NOT_DELETE= 1

Delete EBS volume

Deletion can be executed only if ALL conditions are true

Volume have tag inspected with proper date of inspection

Volume have tag inspected_by with proper juniper id

Volume have tag comment with information that owner was contacted or owner is unknown 

Volume was inspected at least 3 weeks ago

Volume does not have DO_NOT_DELETE tag




**MONITORING**

Impact (Context)

Prometheus core memory reached ~83 GiB against a 100 GiB limit, creating OOM risk and potential monitoring instability. Investigation showed an oversized TSDB head (24M+ series) driven by high cardinality and label churn; top offenders included Flink metrics with run-unique job_id labels.

End-to-End Debug Commands

Run these in order. They are safe, read-only until the final mitigation command.

Check pod memory pressure

k top pods | grep -i prom

Check remote write backlog (rules out queue pressure)

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_remote_storage_pending_samples' | python3 -m json.tool

Check total active head series (main memory driver)

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series' | python3 -m json.tool

Check total head chunks

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_chunks' | python3 -m json.tool

Top metrics by series count

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=topk(20,count+by+(__name__)({__name__!%3D%22%22}))' | python3 -m json.tool

Top jobs by series count

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=topk(20,count+by+(job)({__name__!%3D%22%22}))' | python3 -m json.tool

Check series creation churn rate

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=rate(prometheus_tsdb_head_series_created_total[5m])' | python3 -m json.tool

Validate Flink run-unique label behavior (job_id)

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=count+by+(job_id)(flink_taskmanager_job_task_operator_numRecordsIn)' | python3 -m json.tool | head -30

Check node count sanity for kubernetes_feature_enabled

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=count(count+by(node)(kubernetes_feature_enabled))' | python3 -m json.tool

Immediate mitigation restart

kubectl delete pod -n monitoring prometheus-core-use1prod2-2-green-0

Use when memory is critically near limit and configuration churn reduction is already underway or if you need to quickly clear the head block after labeldrops. Expect short data gap for in-memory head.

Interpreting Results

Remote write backlog: Empty or 0 indicates egress queue is healthy; memory pressure is not due to remote-write buffering.

Head series/chunks: Sustained multi‑million values (e.g., >10M series) point to high cardinality and/or churn. Memory tracks head size closely.

Top metrics/jobs: Large counts highlight where to apply relabeling or aggregation. Flink with run-unique job_id is a common driver.

Series creation rate: High positive rates indicate churn; aim to stabilize via labeldrops or normalization.

Remediation Options

Immediate

Roll Prometheus core pod (command above) to release current head from memory after applying labeldrops.

Temporarily increase memory limit if at imminent OOM risk while fixes roll out.

Reduce scrape interval for high-churn jobs only if safe for SLOs.

Durable

Drop/normalize high-churn labels (e.g., Flink job_id) via metric_relabel_configs.

Aggregate where raw per-run/per-instance detail is not needed (recording rules).

Limit target discovery to necessary endpoints for kubelet/cadvisor/Flink exporters.

Where to change configs

Edit Prometheus scrape and relabeling in the mist-sk8r repository. Navigate by path rather than fragile line anchors:

mist-sk8r/<cluster>/config/flux-monitored/monitoring/prometheus-core/prometheus.yaml

Search within that file for metric_relabel_configs and resources under the Prometheus container to adjust labeldrops and memory requests/limits.

metric_relabel_configs:
  - source_labels: [job_id]
    regex: .+
    action: labeldrop

Validate with promtool check config and peer review before deploy.

Post-Fix Verification Bundle

Run these 5 checks 10–20 minutes after labeldrop rollout and/or a pod restart. Expect a significant drop from prior values.

Head series

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series' | python3 -m json.tool

Head chunks

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_chunks' | python3 -m json.tool

Series creation rate

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=rate(prometheus_tsdb_head_series_created_total[5m])' | python3 -m json.tool

Top jobs by series

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=topk(20,count+by+(job)({__name__!%3D%22%22}))' | python3 -m json.tool

Flink job_id cardinality

kubectl exec -n monitoring prometheus-core-use1prod2-2-green-0 -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=count+by+(job_id)(flink_taskmanager_job_task_operator_numRecordsIn)' | python3 -m json.tool | head -30

RCA Snapshot

Remote write backlog empty → not the driver.

TSDB head very large: series ~24.4M, chunks ~27.4M.

Top single-metric series counts not enough to explain totals.

By-job active series sum ~1.7M; top: Flink taskmanager, kubelet, cadvisor.

Root cause: high time-series cardinality and churn, largely from Flink labels with run-unique job_id.

Escalation and Ownership

Primary channel: #fg_prometheus for Prometheus platform issues.

Coordinate with Flink service owners to validate acceptable labeldrops/aggregation.

If memory remains unstable after labeldrop + restart, page on-call SRE lead and consider temporary memory limit increase while deeper cardinality reduction is implemented.

References

SRE-202807

Alerts related to Prometheus

Prometheus Runbook: Deadman’s Snitch monitoring for prometheus-lag

Prometheus Instances Info – All Envs

Mist Repos



cassandra-reaper

review done (@Shinhye Yun)

 

Single Run

Bring up deployment in new cluster

Scale down in old cluster

DNS switch to the new cluster

Verify UI using http://cassandra-reaper-<env>.mist.pvt/webui 
(Take creds from LastPass saved by name cassandra-reaper)

Check Logs

 

12	
Chomsky

 

Kubernetes

SealedSecret for SFX

Ingress

RoleBindings depend on mist-rbac

ServiceMonitor depends on monitoring

Infrastructure

SignalFX

Elasticsearch

Kafka topic marvis-chat-events-{ENV}

Mist services

PAPI Internal http://papi-internal-{ENV}.mist.pvt

Rasa API

Double run – run the new deployment in parallel

Verify reachability of new deployment using --resolve



curl -vs http://chomsky-production.mist.pvt --resolve chomsky-production.mist.pvt:80:10.5.2.160
switch dns

verify reachability of service using curl -v. it should pass through new LB ip addresses now.

scale down old deployment.



❯ kubens chomsky
Context "gke_mist-k8s-staging_us-west2_gcp-staging-5" modified.
Active namespace is "chomsky".
❯ k get ingress
NAME              CLASS                      HOSTS                      ADDRESS      PORTS   AGE
chomsky-ingress   ingress-nginx-internal-1   chomsky-staging.mist.pvt   10.5.0.116   80      4d12h
❯ curl -vs http://chomsky-staging.mist.pvt/about --resolve chomsky-staging.mist.pvt:80:10.5.0.116
* Added chomsky-staging.mist.pvt:80:10.5.0.116 to DNS cache
* Hostname chomsky-staging.mist.pvt was found in DNS cache
*   Trying 10.5.0.116:80...
* Connected to chomsky-staging.mist.pvt (10.5.0.116) port 80 (#0)
> GET /about HTTP/1.1
> Host: chomsky-staging.mist.pvt
> User-Agent: curl/7.85.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Tue, 14 Feb 2023 11:03:22 GMT
< Content-Type: application/json
< Content-Length: 59
< Connection: keep-alive
<
* Connection #0 to host chomsky-staging.mist.pvt left intact
{"app": "chomsky", "env": "staging", "version": "v0.0.199"}%
@David Kochiashvili (Unlicensed) Please verify once from your end and update the steps if missing any.

13	
centralized-logging

 

Infra:

Image: gcr.io/mist-common-assets/mistsys/centralized-logging-snapshot-manager (GCR pull access)

FluxCD image automation policy: centralized-logging-snapshot-manager

No PVC (stateless; state lives in OpenSearch)

Kubernetes

Deployment snapshot-manager, replicas: 1, serves HTTP on :8080, probes /health

Ingress (ingress-nginx-internal-1, host centralized-logging-snapshot-manager-<env>.mist.pvt) → Service :80 → :8080 (ClusterIP)

ServiceAccount snapshot-manager

RoleBindings depend on mist-rbac (pod-proxy/-exec/-view, subject laputa)

Node pool role=nodes-large, toleration type=nodes-large

Infrastructure

OpenSearch (OPENSEARCH_ENV=<env>) — manages log snapshots/backups

Object storage backend for OpenSearch snapshots (GCS/S3 snapshot repo)

Mist services

Part of the centralized logging stack (log backup lifecycle)

Double-run – run the new deployment in parallel

Verify reachability of new deployment using --resolve

switch dns (centralized-logging-snapshot-manager-<env>.mist.pvt → new LB)

verify reachability using curl -v /health — should pass through new LB IP now

scale down old deployment



❯ kubens centralized-logging
❯ k get pods            # snapshot-manager 1/1 Running
❯ k get ingress
❯ curl -vs http://centralized-logging-snapshot-manager-staging.mist.pvt/health \
    --resolve centralized-logging-snapshot-manager-staging.mist.pvt:80:<new-LB-ip>
< HTTP/1.1 200 OK
# OpenSearch: confirm snapshots still being created/managed (snapshot repo lifecycle healthy)
http://centralized-logging-snapshot-manager-staging.mist.pvt/health


@Kurt Wang @Ranjith Kumar Ravi could you please help verify once from your end.

14	
CFSSL
Client Private CA

KMS Key - (No change required for migration)


Workload Identity Federation (GCP) & IRSA (in case of AWS)

CFSSL pods consume KMS key via GCP service account (in case of GCP) and AWS IRSA (in case of AWS)

Accordingly update new cluster details there in IAM roles/ permissions.


CFSSL Secrets 

Secrets need to be migrated to ESO


CertDB (No change required for migration) 


Connectivity in between K8S pods network and certdb over port 5432 is needed.

GCP - added as authorised network in postgres db).

AWS - Security Group rule needs to be enabled.
(K8S SG → CertDB SG Port 5432)

So accordingly make sure new cluster network is able to connect to CertDB over 5432.

Double run – run the new deployment in parallel

Verify no erros in logs and healthcheck via port-forward:



➜  k port-forward -n cfssl services/cfssl 8888:8888
Forwarding from 127.0.0.1:8888 -> 8888
Forwarding from [::1]:8888 -> 8888
Handling connection for 8888
➜ curl 127.0.0.1:8888/api/v1/cfssl/health
{"success":true,"result":{"healthy":true},"errors":[],"messages":[]}
switch dns

verify reachability of service using curl -v. it should pass through new LB ip addresses now.

scale down old deployment only after QA validation

CFSSL Service Infra Documentation (K8S)
 

 

15	
client-fingerprint

Client Fingerprint

Kubernetes:

Ingress

Sealed secrets / exteranal-secrets

ServiceMonitor depends on monitoring

RoleBindings depends on mist-rbac

Cross service account access mist to nac [Terracode-access]

Infrastructure:

Redis

Dynamo Db table in NAC

Pre-migration:

Verify IRSA is created with correct path - /client-fingerprint/
Ex: arn:aws:iam::<account_id>:role/<env>/client-fingerprint/* this will assure cross account access (mist to nac - terracode-access)

Migration:

Create the new deployment

Validate below:

Check we can reach the NAC ddb from the new irsa <Look for error>

Check nac client-fingerprint health check

Switch DNS to new ingress

Scale down old deployment

Check pod logs for error

Client-fingerprint url Ex: client-fingerprint-<ENV>.mist.pvt should be reachable 

Validate http://client-fingerprint-<ENV>.mist.pvt/health

Need QA to validate the functionality.

Preferred to be migrated by NAC Team.
Review Confirmation (2026)

16	
client-fingerprint-genid-process

Client Fingerprint GenId Process

Kubernetes:

ServiceAccount (plain, no IRSA)

RoleBindings depend on mist-rbac

ClusterIP service on port 8600

Double run: Deploy the new release in the new cluster using the same image tag.

Migrate before or with client-fingerprint. No DNS switch is needed.

When client-fingerprint in the new cluster is healthy, scale the old deployment to 0.

Scale the old deployment backup; it is stateless.

Check pod logs for errors

Check /health on the new pods: kubectl -n client-fingerprint-genid-process port-forward svc/client-fingerprint-genid-process 8600:8600, then curl localhost:8600/health

Validate <http://client-fingerprint-<ENV>>.mist.pvt/health: genid_process and genid_model_server must not be Failed

Need QA to validate the functionality (GenID enrichment).

Preferred to be migrated by the NAC Team, together with client-fingerprint.

Review Confirmation (2026)

17	
client-router (Go)

 

Kubernetes

Ingress

No ServiceMonitor

RoleBindings depend on mist-rbac

Infrastructure

Kafka

Check the health check output

Scale Down old pods

Bring up new pods

Switch DNS

Repeat step '1'



# valide /about endpoint
curl http://client-router-staging.mist.pvt/about
@Sumit Bajaj (Unlicensed) 

18	
consul

 

Consul is used as K/V database. 

Kubernetes

SealedSecrets

Ingress

 

Create service in new cluster 

Create temporary DNS/perform port-forward to consul-ui service kubectl port-forward -n consul service/consul-ui-service 8500:8500

Seed values using temporary ingress
by following 
Migrating Consul K8s Deployment

Verify K/V

Switch DNS 

Scale old service 

Remove temporary DNS 

 Verify that a service can reach consul cluster and populate K/V



# 1. safe to test on papi-integration pod
apt-get update && apt-get install dnsutils --yes
dig consul-cluster-staging.mist.pvt
/usr/bin/consul-template -consul-addr=consul-cluster-staging.mist.pvt:80 -template "/opt/consul/local_settings_consul.ctmpl:/opt/papi/local_settings_temp.py" -once
cd /opt/papi && diff local_settings.py local_settings_temp.py
# 2. Raft Leader elected
curl -s "http://consul-cluster-staging.mist.pvt:80/v1/status/leader"
# Expected: returns IP:port e.g. "100.64.158.199:8300"
# FAIL if empty
# 3. All members alive
curl -s "http://consul-cluster-staging.mist.pvt:80/v1/agent/members" \
  | jq -r '.[] | select(.Status != 1) | .Name'
# Expected: no output (all members Status=1)
# FAIL if any member name is printed
# 4. KV Store has keys
curl -s "http://consul-cluster-staging.mist.pvt:80/v1/kv/?keys&recurse" | jq length
# Expected: non-zero key count
# FAIL if 0 or HTTP non-200
Don’t need QA validation, papi-* service successfully initializing is a validation. 

19	
cruise control

Review required by Kafka/SRE owner

Please define strategy@Konstantin Malko @Suresh Koya @Kurt Wang@Ankit Sambhare @Somesh Prajapati 

Kubernetes: 

Deployment, Service, internal Ingress, ConfigMap.

Infrastructure: Kafka brokers, ZooKeeper.

Confirm no Cruise Control operation is running.

Deploy to the new cluster. 

Validate before switch DNS. 

Freeze rebalance, broker add/remove, demotion, and sampling-control operations.

Scale the old Cruise Control deployment down to 0

Scale the new deployment up to 1.

Validate the new pod.

Switch DNS to the new ingress.

Run final API and mistcli services validation.

Confirm pod and logs are healthy.

Verify /state, /kafka_cluster_state, /load, and /user_tasks return HTTP 200 and valid JSON.

Run mistcli services <environment> cruise-control --yes.

 

Do not automate /rebalance, broker add/remove, broker demotion, sampling control, or proposal-execution control. These require manual approval and post-operation Kafka verification.

20	
demo-simulations

demo-simulations deployment specification
 

Kubernetes: • Namespace: demo-simulations • ConfigMap (PROVIDER, ENV, MIST_ENV, REGION, simulation .properties files) • Service & Deployment (mobilesim container) • No ServiceMonitor / Ingress Infrastructure & Mist Services: • Kafka topic: mobile-client • client-terminator (https://client-terminator-<ENV>.mistsys.net) • PAPI (https://api.mist.com)

Pre-migration:

Create configuration for the new cluster (don’t deploy - the service should not double run).

Migration:

Scale down in old cluster

Scale up in new-cluster

Check the logs (see here for example logs)

Verification process: MIST-166493: [aws-staging-1] C100 demo-simulations
Closed
 

 

21	
dexter

Dexter Deployment Specification
 

Kubernetes:

Ingress

Sealed secrets / external-secrets

ServiceMonitor depends on monitoring

RoleBindings depends on mist-rbac

Cross service account access mist to nac [Terracode-access]

Infrastructure:

PostgreSQL

Dynamo Db table in NAC

Pre-migration:

 Verify IRSA is created with correct path - /dexter/
Ex: arn:aws:iam::<account_id>:role/<env>/dexter/* this will assure cross account access (mist to nac - terracode-access)

Migration:

Create the new deployment

Validate below from the k8s pod logs: <Look for error>

Check we can reach the NAC ddb from the new irsa

Check nac dexter health check

Check postgres connection

Switch DNS to new ingress

Scale down old deployment

Check pod logs for error

Verify the app password from the pod with the one in secretmanager named <dexter_postgres_app>



kubectl exec -it <pod> -n dexter -- env | grep DB_PASSWORD
Dexter url Ex: dexter-<ENV>.mist.pvt should be reachable 

Need QA to validate the functionality.

Preferred to be migrated by NAC Team.

Review Confirmation (2026)

22	
dizzy2 (Go)

Review done

@Somesh Prajapati 

Kubernetes

Ingress

SealedSecret

IPTUN

No ServiceMonitor

RoleBindings depend on mist-rbac

Infrastructure

Redis

Kafka

S3/GS bucket access

External services

GitHub

bring up new deployment

scale down old deployment to 0 immediately after and switch DNS.

 

NOTE: Please add the SA if missing in aws envs

Check in corresponding environment slack channel (i.e. #dizzy-alerts-staging) for alerts

Post DNS switch, run below and monitor #dizzy-alerts-<env> slack channel.



http://dizzy2-staging.mist.pvt/refreshlist
Dizzy2

Dizzy2
 
Dizzy2
  

 

 

23	
flink-app/flink-common

flink-app cluster
 

Kubernetes

No ServiceMonitor

Ingress

RoleBinding (all in flink-app namespace)

Double run flink-common (Zookeeper)

Make sure all Zookeeper pods are running in flink-common namespace

Double run flink-app

Switch flink-app DNS

Scale down old flink-app

Scale down old flink-common

Jenkins job deploy-flink-app-delta to deploy Flink apps in target env

Check if you can reach http://flink-app-<ENV>.mist.pvt/ and Flink apps are running

 

24	
ingest

Review Done @Ankit Sambhare 

Kubernetes: Deployment/Service in ingest namespace, Ingress, ServiceMonitor, RoleBindings depend on mist-rbac

Secret — webhook integration credentials (e.g. MS Teams, Zoom QSS OAuth) — migrated/rotated per-env separately

DNS / Load Balancer: ingest.{ENV}.mist.com → ALB (AWS) / GCLB (GCP)

Shut down old deployment, bring up the new one

"Update the DNS record for ingest.{ENV}.mist.com with ALB (AWS) / GCLB (GCP)..."

After DNS Switch you can rely on the heartbeat to confirm things are good.

If the hearbeat is not working you will get prometheus alerts (in #ingest-alerts). You can also use the following log filter in AWS CloudWatch to confirm the sending/receiving of heartbeats (a heartbeat is triggered every 5m):

Functional check: curl -vs https://ingest.{ENV}.mist.com/about → expect Ingest is a general purpose webhook ingestion service.

For AWS, Go to CloudWatch of the respective env and type “ingest“ in LogInsights, validate the following message using below query



fields @timestamp, @message
| filter @message like /heartbeat/
| limit 20
for GKE, use query in Logs Explorer



resource.type="container"
AND jsonPayload.container_name="ingest"
AND (jsonPayload.message: "heartbeat")
Check service health also in grafana - Service monitor dashboard & Ingest 

 

25	
jenkins

(jenkins in k8s )

airflow2(efs csi driver)

Ingress

nodes-jenkins instance group

Double run jenkins deployment

Make sure jenkins master pod is healthy

Switch jenkins DNS

Scale down old jenkins deployment

steps for jenkins migration:

Make sure IRSA for new cluster is present and other clusters has access to it.
IRSA - https://github.com/mistsys/iac/tree/main/aws-production/init/eks-1-green/iam/jenkins-roleYour team sees richer Github previewsConnect 
Cluster access - https://github.com/mistsys/iac/pull/2951Your team sees richer Github previewsConnect 

Create new deployment in new cluster with following changes:

Update the new IRSA ARN in the serviceaccount.yaml

the serverUrl in configmap should be like below



serverUrl: "https://kubernetes.default.svc"
Reference: 

https://github.com/mistsys/mist-sk8r/pull/46044/files 

(applicable only in aws) IRSA for jenkins created in new cluster in serviceaccount.yaml like https://github.com/mistsys/mist-sk8r/pull/30930/files#diff-24770a671733d4f5b9216d3cb853c8fadb8e634d139b7500dc6da00aeba2e0b9R8 

Make sure that nodepool(s) jenkins-red and/or jenkins-black are created before the deployment configs are merged to new cluster

Once jenkins deployment is running make sure the pod is healthy along with ingress, service, pv and pvc



sbhise@sbhise-mbp aws-staging-1-blue.eks % k get pods -n jenkins
NAME                              READY   STATUS    RESTARTS        AGE
jenkins-master-7785444b6b-p7t5q   1/1     Running   1 (5h32m ago)   5h40m
sbhise@sbhise-mbp aws-staging-1-blue.eks % k get ingress -n jenkins
NAME              CLASS                      HOSTS                           ADDRESS                                                                         PORTS   AGE
jenkins-ingress   ingress-nginx-internal-1   jenkins-blue-staging.mist.pvt   a29e0cee7b3ea4f3f8ebf6671ec1664e-4db4b609b69b97fd.elb.us-east-1.amazonaws.com   80      5h40m
sbhise@sbhise-mbp aws-staging-1-blue.eks % k get svc -n jenkins
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
jenkins   ClusterIP   10.100.75.61   <none>        8080/TCP   5h40m
sbhise@sbhise-mbp aws-staging-1-blue.eks % k get pvc -n jenkins
NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
jenkins   Bound    jenkins   20Gi       RWX            jenkins        <unset>                 5h50m
sbhise@sbhise-mbp aws-staging-1-blue.eks % k get pv -n jenkins | grep jenkins
jenkins                                    20Gi       RWX            Retain           Bound    jenkins/jenkins                                                                 jenkins        <unset>                          5h50m
sbhise@sbhise-mbp aws-staging-1-blue.eks %
4.Before switching ingress make sure the UI is up and all jobs are running and populated as expected using port-forwarding



sbhise@sbhise-mbp aws-staging-1-blue.eks % k port-forward jenkins-master-7785444b6b-p7t5q 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
 

image-20250107-230119.png
 

Switch ingress dns to point to new deployment and once switched verify the dns as below:



sbhise@sbhise-mbp aws-staging-1-blue.eks % dig jenkins-blue-staging.mist.pvt +short
a29e0cee7b3ea4f3f8ebf6671ec1664e-4db4b609b69b97fd.elb.us-east-1.amazonaws.com.
172.31.123.121
172.31.104.93
172.31.7.22
172.31.116.44
Try to run any check-connectivity jenkins job and see if it runs successfully with spawning agent in new cluster namespace of jenkins http://jenkins-blue-staging.mist.pvt/job/check-connectivity/7/console

image-20250107-230131.png
Scaledown deployment in old cluster https://github.com/mistsys/mist-sk8r/pull/30905Your team sees richer Github previewsConnect

 

26	
jcart-connector

 

Kubernetes

Ingress

Infrastructure

Redis

Kong gateway

Shut down old deployment, bring up the new one

Switch the DNS

After DNS switch, 

validate the URL jcart-connector-<env>.mist.pvt
curl -vs http://jcart-connector-<env>.mist.pvt/about

Validate Kong route 



curl http://kong-gtw-admin-<env>.mist.pvt:8001/routes/jcart-connector
 

27	
jsi-term

https://github.com/mistsys/oc-terminatorYour team sees richer Github previewsConnect 

 

Kubernetes

Service type LoadBalancer

ServiceMonitor

RoleBindings

Infrastructure

Redis

Kafka

 

Service is not rolled out across envs, also no updates on NSI page.
Looks service rollout is on hold.

Service is lightweight version of oc-term.

Service verifications are similar to oc-term.
Currently service is running in aws-staging only.

28	
inventory

Mist Cloud Inventory Deployment Specification
 

 

 

 

 

29	
kafka-lag

(pods: 

lag-crawler,

metric-exporter,

redis-master)

review done (@Shinhye Yun)

Kubernetes

SealedSecrets

Ingresses

ServiceMonitor

Monitoring

Prometheus-lag

Alertmanager

PagerDuty

Jenkins pipeline

restarts_prometheus_lag - update jenkins pipeline to point to new cluster - job must be verified working with new k8s cluster. must update cluster in https://github.com/mistsys/jenkins-pipelines/edit/master/config/lags_config/lag-crawler.json 

Single Run

** Get PR review from @Pradeep Kaloori / @Shinhye Yun 

Deploy all three services (lag-crawler, metric-exporter, redis-master) in new cluster

Switch DNS for metric-exporter

Verify that lag metrics are available in Prometheus (metrics to search: lag_time or lag_msg.

Scale down all three services in old cluster.

Check that lag metrics are being populated at metric-exporter’s endpoint (
[Prometheus] prometheus-kafka-lag runbook) 



http://metric-exporter-{ENV}.mist.pvt/metrics/kafka/lag

Check Prometheus/Alertmanager monitoring (Find link to Prometheus-1-2 (lag monitoring) in Environment page)

Check lag metrics are available in prometheus-lag: In prometheus, query lag_time, lag_msg, rate_producer


Make sure alert rules are in place: https://github.com/mistsys/mist-sk8r/tree/master/production-uw1-v121-1.mistk8s.pvt/config/flux-monitored/monitoring/prometheus-2/alertsYour team sees richer Github previewsConnect and check alert rules in Prometheus UI:


Check alerts are valid and being fired to Alertmanager (Find link to Alertmanager in Environment page)

Check metrics in Grafana 

→ If no metrics are shown at metric-exporter’s endpoint,  check in redis (how to)

30	
kafka-ui

review done 

(@Somesh Prajapati )

Kubernetes

Ingress

RoleBinding

Infrastructure

ZooKeeper

Kafka

 

Single run

After connecting to Kafka-ui in new cluster by doing port-forward check that cluster settings showup in the port forwarded instance.

Once done create a PR in IAC to update Kafka-ui  DNS record. 

Repeat step 1 by connecting to kafka-ui with DNS record instead of port forwarding. 

If 1-3 are good. Delete Kafka-ui from old cluster. 

Port forward to kafka-ui and make sure the kafka-ui is up and running.



kubectl port-forward service/kafka-ui -n kafka-ui 9001:8080
Then browse  http://localhost:9001

Check to see kafka cluster settings for existing kafka-ui and see that the same settings show up in new kafka-ui.

Make the DNS change after verification of kafka-ui using port forwarding.

31	
kafka-topic-rate (monitoring)

review done

Infrastructure

Kafka

Double run

Try to keep double run time as small as possible. The topic rate in signalfx and prometheus should show similar message flow rates before and after deployment.

 

32	
kubebox

 

Kubernetes

Ingress

ClusterRoleBinding

ClusterRole

Service

Double run– run the new deployment in parallel

Switch dns (Ingress)

Verify Logs and check if URL is opening.

Scale down old deployment.

Verified that the new pod is in running state. Checked logs and then scaled down old deployment.

Make the DNS change after verification.

33	
Laputa

 

laputa’s service dependencies

papi-internal

papi-internal redis (via predixy)

pubsub redis

kafka

iptun

And before deployment we have some dependencies

mist-rbac

ingress controller (internal)

cert-manager

sealed-secret controller

rbac/annotations in each k8s namespace where we want to support laputa service access (should be all mist app namespaces)

GKE: kube api server proxy support (firewall rule)

we need to generate all the secret again.
reference :- MIST-76188

double run in the new cluster

check secrets,pod,ssl certificate (should be valid), ingress

switch dns to new internal load balancer

verify if you can ssh to pods from laputa and there should be no error in UI. QA to verify much more.

api (make sure you have full kubeconfig generated for the new deployment) 

kubectl get certificate -n laputa

kubectl get secrets (all intended secrets should be present)

open laputa UI - get url per env from environments page. format laputa.<Public_Domain>.com

check the laputa pod logs
We can login and try to validate if we are able to connect pods in laputa UI

@Abhay Rastogi Please verify once from your end and update the steps if missing any.

34	
llm-container

LLM-Container Deployment Specification
 

Infra:

ECR image pull: 660610034966.dkr.ecr.us-east-1.amazonaws.com/mistsys/llm-container — node IAM role needs pull access

No storage bucket / no PVC (stateless)

Kubernetes

ExternalSecret → global-secrets ClusterSecretStore (AWS Secrets Manager) — not SealedSecret

Ingress (ingress-nginx-internal-1, host llm-container-<env>.mist.pvt)

RoleBindings depend on mist-rbac (mist-pod-proxy / -exec / -view; subjects laputa, cauchy2/services)

ServiceMonitor depends on monitoring

Node pool role=nodes-large, tolerations market_type=on-demand, team=mist-large (hard nodeAffinity)

Infrastructure

AWS Secrets Manager (mist/global/agatha/openai*, mist/global/llm-container/*, mist/global/elasticsearch/*)

OpenAI / Azure OpenAI / GPT-4 / JCloud endpoints (egress)

Elasticsearch (ES8)

Mist services

Consumed by laputa, cauchy2 (RBAC + internal calls)

Double-run – run the new deployment in parallel

Verify reachability of new deployment using --resolve

switch dns (llm-container-<env>.mist.pvt → new LB)

verify reachability of service using curl -v — should pass through new LB IP now

scale down old deployment






❯ kubens llm-container
❯ k get ingress
NAME                    CLASS                      HOSTS                            ADDRESS     PORTS
llm-container-ingress   ingress-nginx-internal-1   llm-container-staging.mist.pvt   10.5.0.x    80
❯ curl -vs http://llm-container-staging.mist.pvt/about \
    --resolve llm-container-staging.mist.pvt:80:10.5.0.x
< HTTP/1.1 200 OK
{"app": "llm-container", "env": "staging", "version": "v0.0.1170"}
 

MIST-174021: [aws-production-1] B430 llm-container
Closed
 





jenkins based PP


Prerequisites
Access to mist-sk8r repositories and the Jenkins controller namespaces















 

**jenkins-pipelines
**
mist-sk8r

iac-secrets (for External Secrets)

versions

LastPass access for existing secrets

kubectl access to run rollouts in jenkins namespace

High-Level Workflow
Review existing VM/Docker pipeline and identify dependencies

Create or map credentials to Kubernetes Jenkins

Choose or define a Kubernetes agent

Create the Jenkinsfile under k8s-pipeline

Validate in staging with a test job

Tag and publish via versions repo

Let seed job create production/staging pipelines

Step-by-Step Instructions
1) Review Current Pipeline
Agent config and required tools

Environment variables and secrets

Pre-checks and post actions

Volume mounts and network needs

Credentials usage and scopes

2) Prepare Jenkins Credentials
Ensure secrets used by the pipeline are available to Kubernetes-based Jenkins and backed by LastPass and/or External Secrets.


Copy secret value from LastPass

Load secret into Jenkins ( external secret)

Apply and verify
3) Choose or Create a Kubernetes Agent
Prefer reusing shared agent: inheritFrom 'jenkins-agent'

 

Create/maintain Dockerfile at: mist-cloud/docker-images

Add agent YAML in jenkins-pipelines: k8s-pipeline/agent

Update both AWS and GCP provider folders if custom

Ensure ssh package is installed

Example custom agent YAML: cassandra-agent.yaml

4) Create the Jenkinsfile
Place Jenkinsfile under k8s-pipeline/domain/Jenkinsfile_name. Key rules:

Import shared library at top: @Library('sharedlibs') _

Use pipeline-level Kubernetes agent

Add a Pre-check stage as the first stage to guard against seed-triggered runs

Use shallow checkout helper and proper credentials

Add reasonable options: buildDiscarder and timeout

Send custom metrics and clean workspace in post


Hello World example (ready to copy)

Migrating from Docker agent to Kubernetes agent
5) Stage-Specific Containers (Optional)
If your agent pod defines multiple containers, run a stage in a specific container:



stage('Run in cassandra container') {
  steps {
    container('cassandra-container') {
      sh 'echo "Executing inside cassandra-container"'
    }
  }
}
Example multi-container agent: cassandra-agent.yaml

6) Validate in Staging
Branch your changes and name the Jenkins job with test- prefix in sandbox view to suppress alerts

Run the pipeline on staging and observe runtime, logs, and resource usage

For cron pipelines, run for at least one day and verify controller CPU/memory/disk headroom

7) Publish and Seed
Create PR to jenkins-pipelines; after review and merge, create a new tag

Update the job entry in the versions repo and include these fields:

name (pipeline name)

tag (e.g., refs/tags/v1.0.X)

pipeline_path (k8s-pipeline/.../Jenkinsfile_...)

priority (required alert classification label)

view (folder/view name)

Allow the next seed run to create/update jobs per versions intent

Priority required: When creating or updating Jenkins jobs in the versions repo, define a priority label for each job. If a job has no priority label, it will not match these alert rules unless it is separately classified.

Priority alerting policy: P0 is for production-critical jobs and alerts on the first failure within 10 minutes. P1 is for important jobs and alerts after 2 or more failures within 10 minutes. P2 is for lower-priority jobs and alerts after 2 or more failures within 24 hours, for example snapshot Jobs that runs for 3 times in 24 hrs.

Priority is read from the jenkins_job_status metric label, for example priority="P0", priority="P1", or priority="P2".


Sample versions entry
Best Practices
Keep build history lean: set buildDiscarder based on pipeline frequency

Set timeout slightly above historical average runtime (e.g., 15–20 min job → 1 hour timeout)

Pin branches or tags for deterministic builds; prefer shallow clones to reduce runtime

Centralize pipeline config under k8s-pipeline/config; avoid legacy jenkins-job-config repo

Ensure all custom agents include ssh and needed CLIs

Separate sensitive values into Jenkins credentials; do not hardcode secrets

Troubleshooting

Credentials not found in Jenkins

Agent pod fails to start

Git clone failures
Definition of Done

Secrets mapped from LastPass to External/Sealed Secrets and loaded via mist-sk8r

CASC updated and Jenkins restarted to load credentials

Agent YAML selected/created and validated for required tools

Jenkinsfile created under k8s-pipeline with sharedlibs import, Pre-check, options, environment, and post

Staging test job (prefixed with test-) executed successfully

versions repo updated with correct tag and pipeline_path

Seed run completed and jobs created in target environments

