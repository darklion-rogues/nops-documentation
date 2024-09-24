# Proactive Cost Controls

In addition to providing visibility into costs and how they are allocated, nOps is also able to take proactive steps to prevent some types of cost overruns before they occur.

Kubernetes deployments and their pods consume resources on the underlying nodes on which they run. nOps is able to monitor this usage, down to the container level, and calculate how much a specific deployment out of many cost based on a variety of factors. In order to monitor usage, something must exist to monitor. Rather than allowing some deployments to be created and consume excess money, they can be blocked. For example, you as a nOps administrator may wish to restrict the `ticketing` namespace to a monthly allowable budget of $500. Once that budget has been consumed, you wish to prevent any further deployments into that namespace as these will cause a cost overrun.

Using a combination of the [predictions API](/apis/governance-apis/spec-cost-prediction-api.md) and [budgets](/using-nOps/navigating-the-nOps-ui/budgets.md), nOps is able to enforce budgets with the addition of a dynamic admission controller such as [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/) or [Kyverno](https://kyverno.io). In this process, a Kubernetes admission controller is installed in the cluster along with a policy which asks nOps for cost predictions and verifies those against a matching budget. The decision to allow or deny then rests with the admission controller.

{% hint style="info" %}
When a budget used for these purposes is configured with a grouping other than "Cluster", it should be confined to a single-cluster nOps installation.
{% endhint %}

For more details on this process and how to achieve proactive cost controls with a step-by-step guide using the Kyverno policy engine, please refer to the nOps blog post [here](https://blog.nOps.com/blog/kyverno-and-nOps/#proactive-budget-control).
