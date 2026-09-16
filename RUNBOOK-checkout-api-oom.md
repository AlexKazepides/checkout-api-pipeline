Runbook: checkout-api OOMKilled Incident

Applies to: checkout-api Trigger: Pods restarting with Reason: OOMKilled, intermittent request failures Related postmortem: checkout-api memory-limit incident (2 min MTTD, 12 min MTTR)
1. Detect

Signs you're dealing with this scenario:

    Alerting or dashboards show a climbing restart count for checkout-api
    Users or upstream services report intermittent connection errors (not malformed responses)
    kubectl get pods shows increasing RESTARTS for checkout-api pods

kubectl get pods -l app=checkout-api -o wide

Look at the RESTARTS column. If it's climbing across multiple pods over a short window, move to Diagnosis.
2. Diagnosis
2.1 Confirm OOMKilled is the actual cause

Don't assume — a restart can come from a crash, a failed liveness probe, or an OOM kill. Confirm it explicitly:

kubectl describe pod <pod-name> -n <namespace>

Check the Last State section of the container status. You're looking for:

Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137

If you see OOMKilled, proceed. If you see a different reason (e.g. Error, CrashLoopBackOff from an app-level panic), this runbook doesn't apply — investigate application logs instead.
2.2 Check current resource configuration

kubectl get deployment checkout-api -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].resources}'

or more readably:

kubectl describe deployment checkout-api -n <namespace> | grep -A5 "Limits\|Requests"

Note the current memory limit. Compare it against what you'd expect the workload to need — a limit that looks implausibly low (e.g. single-digit Mi for a service handling checkout traffic) is a strong signal of a misconfigured limit rather than a genuine leak.
2.3 Check actual observed memory usage

kubectl top pods -l app=checkout-api -n <namespace>

If metrics-server data isn't available or you need history, check your metrics backend (Prometheus/Grafana/Datadog) for the container's memory usage over the last few hours, before the incident window. This tells you whether the limit is simply too low for normal load, or whether usage itself is climbing unusually (which would instead suggest a leak).
2.4 Check recent changes

kubectl rollout history deployment/checkout-api -n <namespace>

Correlate the OOMKilled onset time with any recent deploys or config changes (resource limits, replica counts, traffic shifts). A resource-limit change applied without checking current usage data is a common root cause — confirm whether that happened here.
2.5 Decide: bad limit vs. real leak

    Limit set too low relative to steady-state usage (usage was stable before the change, drop coincides with a config change) → go to Resolution 3.1.
    Usage was climbing over time even under the old limit, or usage is unbounded/growing per-request → this may be a genuine memory leak. Go to Resolution 3.2.

3. Resolution
3.1 Revert or correct the memory limit (misconfigured limit)

Find the last known-good value from rollout history or version control, then patch it:

kubectl set resources deployment/checkout-api -n <namespace> \
  --limits=memory=<known-good-value> \
  --requests=memory=<known-good-value>

Example, reverting to a previous 128Mi limit:

kubectl set resources deployment/checkout-api -n <namespace> \
  --limits=memory=128Mi \
  --requests=memory=128Mi

Alternatively, if the change came from a manifest/Helm values file, revert it there and reapply:

kubectl rollout undo deployment/checkout-api -n <namespace>

Then confirm the rollout completes and pods stabilize:

kubectl rollout status deployment/checkout-api -n <namespace>
kubectl get pods -l app=checkout-api -n <namespace> -w

Watch RESTARTS — it should stop climbing. Confirm requests are succeeding via your usual health check or synthetic monitor.
3.2 Mitigate a suspected real leak

If usage genuinely grows unbounded:

    Raise the memory limit as a short-term mitigation to stop the bleeding:

    kubectl set resources deployment/checkout-api -n <namespace> --limits=memory=<higher-value>

    Capture a heap/memory profile from a live pod if your runtime supports it, before killing it, for later investigation.
    File a follow-up ticket for engineering to investigate the leak — this runbook only covers stabilizing production, not root-causing application-level memory growth.

4. Verify resolution

kubectl get pods -l app=checkout-api -n <namespace>
kubectl top pods -l app=checkout-api -n <namespace>

    RESTARTS count is flat for all pods over a sustained window (e.g. 10+ minutes)
    Memory usage is stable and comfortably under the new limit
    Error rate / request success rate back to baseline

5. Follow-up

    Confirm the resource-limit change (if that was the cause) goes through review with actual usage data attached before being reapplied.
    If no automated guardrail exists to reject implausible resource values, file a ticket to add one (e.g. a policy check or admission webhook flagging limits far below historical usage).
    Update this runbook with any details specific to this incident that future responders would find useful.

