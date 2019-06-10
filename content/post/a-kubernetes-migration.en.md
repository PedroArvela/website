---
title: "A Kubernetes Migration"
author: "Pedro Arvela"
type: "post"
date: 2018-10-01
tags: []
---

*This was originally posted in the [Unbabel R&D Medium](https://medium.com/unbabel) with the title [Unbabel migrated to Kubernetes and you won’t believe what happened next!](https://medium.com/unbabel/unbabel-migrated-to-kubernetes-and-you-wont-believe-what-happened-next-b39f082def1c)*

<!--more-->

---

With great growth comes a great set of challenges, and a great responsibility to meet them.

Here at Unbabel, our exponential growth has seen demand grow, and with it, the size and number of projects we create.

Historically, we have used Amazon, keeping one project per EC2 instance, where each project is configured through Ansible playbooks tailored for them. Until not too long ago, this seemed to be working reasonably well. Each project had an instance and the playbook would make sure the project and its dependencies were kept up to date.

{{% figure src="/img/post/2018/10/ansible.png" caption="Every single project had an Ansible playbook" %}}

However, this was not without its shortcomings. As a project’s use grew, how should we properly split the load of that over multiple instances? How would we make sure the instances were not only up to date, but also consistent on versions, dependencies and configurations? As changes become more frequent, could we really expect a developer to run an Ansible playbook (that might potentially take a few minutes) on every single instance where that project is?

All of these questions have possible solutions and all of these issues have possible workarounds, but all of them amount to more cognitive overhead for our developers.

Furthermore, as our teams grew and new projects appeared, keeping a single project per instance started to become wasteful. Often, some smaller projects could leave an instance idle most of the time, only needing compute power at specific times.

The amount of instances multiplying was also clogging up our DevOps team. Having to configure and maintain logging, metrics and keeping various other configurations up to date may have made sense with a dozen or so machines — but when you reach hundreds, it’s not at all scalable.

## Time for a change

At the start of 2018, Unbabel’s DevOps team set out to solve these rising issues. Our goal was to make deploys faster and easier for developers, reduce the time it takes to scale a project’s resources up or down, and automate as much as possible any extra configurations for logging, application metrics, and resource control. Ideally, even make them invisible to the projects themselves.

After some investigation, [Kubernetes](https://kubernetes.io/) seemed to offer what we were after:

- It’s tailored for both tens to thousands of micro-services.
- Its deployments run isolated from each other — it provides low- coupling of services.
- Its deployments run isolated from the base instance — it- abstracts the underlying system.
- Updates on deployments take seconds, not minutes.

These and many other advantages made it a very strong candidate for these changes.

{{% figure src="/img/post/2018/10/kubernetes.png" caption="Kubernetes’ offers made it a very strong candidate" %}}

However, not everything was ideal. Kubernetes’ complexity and no-batteries-included ideology meant that there were several hurdles we’d have to go through if we wanted to make this migration happen.

We’d have to set up logging and metrics monitoring ourselves, and discover how to automate service discovery for these when applicable. The fact that Kubernetes is so broad also means that it has a high learning curve, potentially slowing adoption by our teams. Most of our projects are also not simple micro-services, and that makes updating a deployment more complex in a few cases.

## Putting together a plan

With our goals in sight and possible hazards in mind, we set out to define a plan of migration. The idea was simple, start with projects with low traffic, low risk, and that would most benefit from migrating to Kubernetes. These would give us a safety buffer that would allow us to dip our toes in without extreme compromise, as well as demonstrating to Unbabel’s engineering teams the advantages of the new system’s improved workflow.

We set up a cluster in Google Cloud and started migrating the first staging environments. We set up DaemonSets for logging and set up a Prometheus cluster inside for monitoring. We experimented with RBAC and namespaces so team members could have access to their projects and could easily administer anything necessary.

To help our devs get used to the new environment, we prepared some workshops they could attend on getting started with Kubernetes, created guides on how to set up their environments, and had some hands-on tutorials where they could launch their own simple mock services inside a minikube cluster. Additionally, we helped answer every single question that was raised, and tried to mitigate common questions with more comprehensive guides and better tooling.

Naturally, some unexpected situations did occur, where expectations did not match reality, and changes to the environment had to be done as we switched our approach. Even so, projects there still had a better environment than the previous projects using Ansible playbooks, and once configured, the maintenance burden on these was much lower, reducing our toil.

As time went on we started suggesting to owners of new projects to set up their environments directly on Kubernetes, as well as for existing projects with major migrations incoming to do the same.

## So, how did it go?

We currently have four Kubernetes clusters, having migrated many projects there, and are in the process of migrating a few of our core services. As a result, the number of individual Ansible playbooks in use is on the decline.

Of course there were some problems along the way. For example, Kubernetes’ secrets are a big hassle to manage. The Kubernetes Dashboard is also extremely flimsy to set up if you want to have both the ease of login using OICD, as well as having each developer use their own account to prevent possible privilege escalation on the dashboard. We even got hit with the 5–15s DNS lookups problem as one of the major services was migrated.


{{% figure src="/img/post/2018/10/kubernetes-dashboard.png" caption="The (in)famous Kubernetes dashboard" %}}

But even with these problems, we are really happy with the results. Setting up a new project went from something that could take hours or days to something that can be done in minutes, and most of the time we are talking about projects that can be almost completely self-serviced by the owners themselves.

Manual instance maintenance tasks are on the decline, and instead of focusing on updating service X, Y or Z on all the instances, one at a time, we can focus on higher-level problems, such as “how can we further improve our monitoring infrastructure?” or “how can we automate task X?”. Instead of manually implementing blue-green deployments or having to deal with downtime windows, Kubernetes’ rolling updates solved that right away for us.

A few perks that weren’t expected also became great additions to our development practices. By integrating Kubernetes deploys in our CI pipeline, we got amazing reproducibility and easy rollbacks for any commit we make. Developers in general were able to stop worrying about deploying new versions, and let CI do that for them. Any changes to the dev and production environments became absolutely documented, and any regression could be pinpointed down to the exact commit that was deployed thanks to annotations on our Grafana dashboards.

{{% figure src="/img/post/2018/10/grafana-dashboard.png" caption="Our Grafana dashboards are indeed amazing" %}}

One special success case was our core monolith, that previously had a slow and error-prone deploy that could take up to 20 minutes to deploy, built upon fragile fabric scripts that had the tendency to leave things in inconsistent state if anything was even slightly off the expected. We set up a CI stage to build a docker image, moved as many processes from deploy-time to compile-time (*I’m looking at you, Django’s collectstatic*), set up a staging environment, and after much preparation set up a switchover. Everything went well, and our deploys now take less than 1 minute to run, all through CI!


## What’s next?

Making this change was one of the biggest jumps in our journey, but that doesn’t mean we are done. We still have a lot of projects that need to be migrated to Kubernetes, including a couple of major internal projects.

Additionally, we still want to focus on improving the quality of life of developers. We want to actually find and enforce a good solution to secret management that prevents developers from having to convert to and back from base64, as well as keeping those secrets safe and versioned.

The Kubernetes dashboard is still a major hurdle for us and we are investigating whether we should have users login using OICD to their accounts, or to find an altogether different alternative that can provide us with a good interface and reasonable security.

Finally (for now), many deployments have a similar structure and much of their creation could be automated and simplified. This could possibly be done by first detecting common dependencies such as databases and event queues and creating central infrastructure that could be used, as well as detecting the common patterns of projects and create helm charts that follow those.

## Wrap it up

We started this journey with a need to scale our resources and to simplify our deployments. In the end, we (er) ended up with not only that, but a declarative and continuous deployment pipeline, better security, monitoring and scalability.

It certainly wasn’t easy and required cross-functional work with the entire dev team, but the benefits we gained surpassed all of the hurdles faced along the way. And our work is far from done. There are deployments to migrate! Pipeline improvements to be made! The world doesn’t stop evolving around us, and neither do we.