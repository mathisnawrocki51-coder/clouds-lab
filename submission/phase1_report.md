# Phase 1 — Comparison Report

Mathis Nawrocki-Lachapelle — EURECOM, Cloud Computing, Phase 1 (echo bot).
Everything was done from Google Cloud Shell, project `eurecomgpt-mathis`, region `us-central1`.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

(`SIZE: 49987399` bytes in Artifact Registry. This is the compressed size stored in the registry, so it is smaller than what `docker images` would show locally. The multi-stage build on `python:3.11-slim` keeps it far below the 400 MB target.)

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

The VPC is in custom mode with a single subnet `10.10.0.0/24` in `us-central1`. I added two ingress firewall rules, `allow-echo-8080` (tcp:8080) and `allow-echo-ssh` (tcp:22), both with `--target-tags=echo-http`. The tag is needed because a rule with a target tag only applies to the VMs that carry this tag. Without it, the rule would apply to every VM in the network, so the tag lets me open ports 8080 and 22 only on `echo-vm`. A custom VPC starts with "deny all ingress", so without the SSH rule `gcloud compute ssh` would just time out. I used source `0.0.0.0/0` because the Cloud Shell IP changes and the GitHub CI must reach port 8080. This is OK for a VM that lives a few hours, but it would not be acceptable for a real service.

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->250.1<!--/answer--> | <!--answer:vm_warm_median_ms-->211.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1964.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->636.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->859.6<!--/answer--> | <!--answer:function_warm_median_ms-->635.4<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Output of `measure.py` (one single run with the 3 URLs, after about 22 min without any request):

~~~
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  250.1ms  warm_median=  211.9ms
            #############
[OK  ] cloudrun  cold= 1964.1ms  warm_median=  636.1ms
            ########################################
[OK  ] function  cold=  859.6ms  warm_median=  635.4ms
            ########################################
------------------------------------------------------------

Wrote submission/phase1_report.json — commit it and push.
~~~

Warm samples (20 per platform), min / median / p95 / max in ms:

| Platform | min | median | p95 | max |
|---|---|---|---|---|
| VM | 210.7 | 211.9 | 214.7 | 215.3 |
| Cloud Run | 121.6 | 636.1 | 683.3 | 687.9 |
| Function | 119.8 | 635.4 | 680.8 | 683.1 |
<!--/answer-->

**What I observed.**
- **VM:** the cold value (250 ms) is almost the same as the warm values. The container never stops, so there is no real cold start. The first request is only a bit slower, probably because of connection setup.
- **Cloud Run and Function:** the cold request is clearly slower (1964 ms and 860 ms) because a new container had to be started. To make sure the instances were really at zero, I checked Cloud Logging (`STARTUP TCP probe succeeded`): a new instance of `echo-bot` started at 09:25:22 UTC and a new instance of `echo` at 09:25:34 UTC, exactly during the measurement.
- **Warm values:** the warm samples of both HTTPS services form two groups, around 125 ms and around 640 ms. The VM (plain HTTP) stays flat at about 212 ms. `measure.py` opens a new connection for each request and my Cloud Shell is far from `us-central1`, so I think most of the time is network and HTTPS/TLS setup, not my Flask code. I am not sure why there are exactly two groups. My guess is that sometimes the connection setup through Google's front end is faster, but I did not verify this.
- I only have **one** cold sample per platform, so the cold numbers are an order of magnitude, not a precise result.

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
**VM:** nothing is created automatically. There is one `e2-micro` (2 shared vCPUs, 1 GB RAM) running gunicorn with 1 worker and 8 threads, so at most 8 requests are handled at the same time and the others wait in a queue. The limit is the machine size I picked. To scale, I would have to create more VMs and a load balancer myself.

**Cloud Run:** new container instances (copies of my image) are started automatically when the load increases, and they go back to zero when idle. Each instance can take several requests at once (concurrency, 80 by default), and the hard limit is my flag `--max-instances=3`.

**Function:** same mechanism (new instances of the function container), but `maxInstanceRequestConcurrency: 1`, so each instance takes only one request. For 100 requests at once it needs many instances. I did not pass a max flag, so the limit is Google's default for gen 2 functions.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
**VM (most work):** I had to create the VPC, the subnet and two firewall rules, give the VM's service account the `artifactregistry.reader` role, create the VM with the right tag, SSH in, install Docker, configure Docker authentication and run the container. After that, I am responsible for the OS updates, restarts and security, and the external IP costs money even when nobody uses it.

**Cloud Run:** one `gcloud run deploy` command, but it needs a ready image, so I had to write the Dockerfile and build it with Cloud Build. My first build failed with a 403 because the default compute service account could not read the Cloud Build bucket, and I fixed it by adding the `cloudbuild.builds.builder` role. Nothing to maintain after that.

**Function (least work):** one `gcloud functions deploy` command with the source folder. No Dockerfile and no image to manage: Google built the container itself (about 40 s).
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
~~~
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-104233528038.us-central1.run.app
LAST DEPLOYED BY: service-104233528038@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-14T09:02:42.802469Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-104233528038.us-central1.run.app
LAST DEPLOYED BY: mathis.nawrocki-lachapelle@eurecom.fr
LAST DEPLOYED AT: 2026-10-14T09:01:38.577608Z
~~~
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The list shows **two** Cloud Run services, not one. `echo-bot` was deployed by me, and `echo`, my Cloud Function, was deployed by `gcf-admin-robot`. So a gen 2 Cloud Function is actually a Cloud Run service. When I deployed it, Google built a container from my source code with Cloud Build and buildpacks, stored it in the `gcf-artifacts` repository, and deployed it on Cloud Run for me.

Because both use the same infrastructure (same HTTPS front end, same scale-to-zero, same way of starting containers), their warm medians are almost identical (636.1 vs 635.4 ms) and both have a real cold start, unlike the VM. In my single run the Function cold start was lower (860 vs 1964 ms), but with only one sample I cannot say that one is really faster.

What really differs is the packaging and the default settings. For Cloud Run I wrote the Dockerfile and chose gunicorn and the flags. For the Function, Google chose the image and the server (functions-framework) and applied smaller defaults: 256 MB of memory, 0.17 vCPU and a concurrency of 1.
<!--/answer-->
