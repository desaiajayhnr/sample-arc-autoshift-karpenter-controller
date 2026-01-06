# zonal-autoshift-karpenter
This is a concept showing how you can use zonal autoshift's integration with EventBridge to automatically remove zones from a Karpenter node pool when an autoshift is underway.

## High level setup instructions
1. Create an EventBridge rule for zonal shift, "Autoshift in progress" events.
2. Configure the rule to send autoshift events to an SQS queue.
3. Create an IAM role for the zonal-autoshift-karpenter pod to assume a role that allows it to read messages from the queue. If using IRSA, add the roleArn to the pod's service account. If using Pod Identity, create a pod identity association. 
4. Apply the zonal-autoshift-karpenter deployment.yaml

## Do-Not-Disrupt Feature

The application supports an optional feature to label and cordon nodes in the impaired availability zone during an autoshift event. This prevents Karpenter from disrupting workloads on those nodes while they're in the impaired zone.

### How it works:
1. **During Autoshift**: When an autoshift event occurs, the application will:
   - Get all nodes in the impaired AZ
   - Apply the `karpenter.sh/do-not-disrupt=true` label to each node
   - Cordon the nodes (mark as unschedulable)
   - Then proceed with the normal nodepool modification

2. **After Autoshift**: When the autoshift completes, the application will:
   - Find all nodes with the `karpenter.sh/do-not-disrupt=true` label in the restored AZ
   - Remove the label
   - Uncordon the nodes (mark as schedulable again)
   - Then proceed with the normal nodepool restoration

### Enabling the feature:

To enable this feature, uncomment the `args` section in the `deployment.yaml`:

```yaml
containers:
  - name: karpenter-sqs-subscriber
    image: $AWS_ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/zonalautoshiftkarpenter
    args:
      - "--enable-do-not-disrupt"
```

Alternatively, you can use the short form:
```yaml
args:
  - "-d"
```

By default, this feature is **disabled**. The application will only modify nodepools without touching individual nodes.
