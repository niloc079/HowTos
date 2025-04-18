# Comprehensive Guide: Upgrading Kubernetes in Dev and Production Environments

- *Potentialy outdated per AI
- This guide provides a detailed, step-by-step approach for upgrading Kubernetes clusters in both development and production environments, with a focus on minimizing risk and ensuring operational continuity.

## Pre-Upgrade Phase

1. **Perform a cluster health assessment**
   - Run `kubectl get nodes` to verify all nodes are in Ready state
   - Execute `kubectl get pods --all-namespaces` to check for any pods in CrashLoopBackOff or Error state
   - Use `kubectl describe nodes` to identify any resource constraints
   - Run `kubectl get componentstatuses` to verify control plane component health

2. **Document current cluster configuration**
   - Create a comprehensive inventory of all workloads using `kubectl get all --all-namespaces -o yaml > cluster-inventory.yaml`
   - Document custom resources with `kubectl get crd -o yaml > custom-resources.yaml`
   - Catalog all storage classes with `kubectl get sc -o yaml > storage-classes.yaml`
   - Export all ConfigMaps and Secrets (safely) for reference

3. **Check Kubernetes version compatibility matrix**
   - Verify current version with `kubectl version --short`
   - Consult official Kubernetes version skew policy documentation
   - Check compatibility with your CNI, CSI, and other critical components
   - Review release notes for known issues in target version

4. **Update cluster management tools**
   - Update kubectl client: `sudo apt-get update && sudo apt-get install -y kubectl`
   - Update kubeadm if used: `sudo apt-get update && sudo apt-get install -y kubeadm`
   - Ensure version compatibility between client tools and target cluster version

5. **Create a detailed upgrade plan document**
   - Define upgrade sequence (dev → test → staging → production)
   - Establish communication channels for stakeholders
   - Document rollback procedures and decision criteria
   - Set maintenance windows and user notification schedules

6. **Perform comprehensive backup**
   - Back up etcd: 
     ```
     ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 \
       --cacert=/etc/kubernetes/pki/etcd/ca.crt \
       --cert=/etc/kubernetes/pki/etcd/server.crt \
       --key=/etc/kubernetes/pki/etcd/server.key \
       snapshot save snapshot.db
     ```
   - Verify etcd backup: 
     ```
     ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshot.db
     ```
   - Export all Kubernetes resources: 
     ```bash
     for n in $(kubectl get -o=custom-columns=NAMESPACE:.metadata.namespace,KIND:.kind,NAME:.metadata.name \
       pv,pvc,configmap,ingress,service,secret,deployment,statefulset,hpa,job,cronjob \
       --all-namespaces | grep -v 'NAMESPACE' | awk '{print $1","$2","$3}'); do 
         IFS=',' read -ra ADDR <<< "$n"
         kubectl get -n ${ADDR[0]} ${ADDR[1]} ${ADDR[2]} -o yaml > "backup/${ADDR[0]}-${ADDR[1]}-${ADDR[2]}.yaml"
     done
     ```
   - Backup persistent volumes data if applicable

7. **Set up enhanced monitoring**
   - Configure additional monitoring alerts for cluster components
   - Set up dashboard for real-time visibility during upgrade
   - Establish baseline metrics for post-upgrade comparison
   - Enable verbose logging for critical components

## Development Environment Upgrade

8. **Create a clone of your development environment**
   - Spin up a temporary cluster matching your dev environment
   - Deploy representative workloads to this temporary cluster
   - Use this environment for upgrade testing

9. **Perform a dry run upgrade**
   - With kubeadm: `sudo kubeadm upgrade plan`
   - Review the upgrade plan output carefully
   - Address any warnings or prerequisites
   - Document expected changes

10. **Upgrade control plane components in dev**
    - Apply upgrade using kubeadm: `sudo kubeadm upgrade apply v1.XX.Y`
    - Upgrade kubelets on control plane nodes
    - Restart control plane components: `sudo systemctl restart kubelet`
    - Verify control plane health: `kubectl get pods -n kube-system`

11. **Upgrade worker nodes in dev (one at a time)**
    - Cordon node: `kubectl cordon nodeName`
    - Drain workloads: `kubectl drain nodeName --ignore-daemonsets --delete-emptydir-data`
    - Upgrade kubelet: `sudo apt-get update && sudo apt-get install -y kubelet=1.XX.Y-00 kubectl=1.XX.Y-00`
    - Restart kubelet: `sudo systemctl daemon-reload && sudo systemctl restart kubelet`
    - Uncordon node: `kubectl uncordon nodeName`
    - Verify node status: `kubectl get nodes` and `kubectl describe node nodeName`

12. **Update CNI and add-ons in dev**
    - Check for CNI version compatibility with target Kubernetes version
    - Apply CNI update: `kubectl apply -f <cni-manifest-for-new-version>`
    - Update storage providers if needed
    - Update service mesh components if deployed

13. **Run comprehensive tests in dev**
    - Execute end-to-end application tests
    - Verify all services and ingresses are accessible
    - Test pod scheduling and auto-scaling
    - Validate persistent volume operations
    - Check custom resource handling

14. **Document lessons learned from dev upgrade**
    - Note any unexpected behavior
    - Document troubleshooting steps for issues encountered
    - Record time taken for each upgrade step
    - Update production upgrade plan with lessons learned

## Production Environment Upgrade

15. **Communicate the upgrade plan to stakeholders**
    - Send detailed notification with timeline
    - Share expected downtime for each service
    - Provide escalation contacts during the upgrade
    - Conduct pre-upgrade briefing with operations team

16. **Scale up critical infrastructure**
    - Add additional capacity if needed to handle transient load
    - Ensure adequate resources for running workloads on fewer nodes
    - Consider temporarily scaling up control plane if using managed Kubernetes

17. **Upgrade Kubernetes API resources to compatible versions**
    - Update Deployments, StatefulSets, DaemonSets to use APIs that won't be deprecated
    - Check for deprecated APIs: `kubectl get --raw /metrics | grep "apiserver_requested_deprecated_apis"`
    - Run `kubectl convert` to update manifests if needed

18. **Pre-pull container images on all nodes**
    - Identify all images needed by the updated control plane
    - Pre-pull images on all nodes to avoid image pull delays
    - Verify image availability: `crictl images list`

19. **Set up maintenance window and user notifications**
    - Activate maintenance page for user-facing applications if needed
    - Enable traffic routing to backup systems if available
    - Start detailed activity logging

20. **Perform staggered control plane upgrade**
    - In HA setups, upgrade one control plane node at a time
    - Verify etcd cluster health between each node upgrade
    - Monitor API server availability throughout
    - Check `kubectl version` after each control plane node is upgraded

21. **Validate control plane health before proceeding to worker nodes**
    - Verify all control plane components are running: `kubectl get pods -n kube-system`
    - Check API server health: `kubectl get componentstatuses`
    - Verify etcd cluster health: `ETCDCTL_API=3 etcdctl endpoint health --cluster`
    - Test core functionality: create a test pod, service, etc.

22. **Implement progressive worker node upgrades**
    - Group nodes into batches (e.g., 20% of fleet at a time)
    - Implement node selectors/taints to control workload placement during upgrade
    - For each batch:
      - Cordon all nodes in batch: `kubectl cordon nodeName`
      - Drain workloads properly: `kubectl drain nodeName --ignore-daemonsets --delete-local-data --force`
      - Upgrade kubelet and related components
      - Restart services: `sudo systemctl daemon-reload && sudo systemctl restart kubelet`
      - Verify node status before uncordoning: `kubectl get nodes` and `kubectl describe node nodeName`
      - Uncordon nodes: `kubectl uncordon nodeName`
      - Run validation tests before proceeding to next batch

23. **Upgrade cluster networking components**
    - Apply CNI updates according to vendor documentation
    - Verify network policy enforcement
    - Test pod-to-pod and pod-to-service communication
    - Validate network performance metrics

24. **Upgrade supporting components**
    - Update metrics-server
    - Upgrade cluster autoscaler
    - Update CSI drivers and storage provisioners
    - Upgrade service mesh if deployed

25. **Run production validation tests**
    - Execute smoke tests for all critical applications
    - Verify horizontal pod autoscaling functionality
    - Test pod disruption budgets
    - Validate ingress controllers and load balancers
    - Check monitoring system data collection

26. **Verify resource limits and requests**
    - Check if default resource limits have changed
    - Adjust resource quotas if needed
    - Verify namespace limits are appropriate for the new version

27. **Post-upgrade cleanup**
    - Remove deprecated API usage
    - Clean up any temporary resources created during upgrade
    - Update documentation with new version information
    - Archive logs and upgrade artifacts

28. **Monitor cluster for 24-48 hours post-upgrade**
    - Watch for any instability in workloads
    - Check for unexpected API server errors
    - Monitor resource usage patterns
    - Verify automated scaling and healing functions

29. **Conduct post-upgrade review**
    - Document lessons learned
    - Update runbooks with new troubleshooting procedures
    - Share upgrade experience with team members
    - Plan for improvements in next upgrade cycle

30. **Update disaster recovery procedures**
    - Ensure backup procedures are compatible with new version
    - Test restoration procedures on the upgraded cluster
    - Update DR documentation with version-specific information

## Additional Azure Kubernetes Service (AKS) Specific Steps

31. **Check AKS version availability**
    - Use `az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster` to see available versions
    - Review AKS-specific release notes for the target version

32. **Create a snapshot of the AKS cluster**
    - Use Azure Backup for AKS or create a custom backup solution
    - Export all custom resources and configurations

33. **Plan for managed add-on upgrades**
    - Identify which AKS add-ons will automatically upgrade
    - Plan manual upgrades for any non-managed components

34. **Execute AKS control plane upgrade**
    - Use Azure CLI: `az aks upgrade --resource-group myResourceGroup --name myAKSCluster --kubernetes-version 1.XX.Y`
    - Or use Azure Portal to initiate the upgrade

35. **Monitor AKS upgrade progress**
    - Watch upgrade status: `az aks show --resource-group myResourceGroup --name myAKSCluster --output table`
    - Monitor through Azure Portal
    - Check node pool status through `kubectl get nodes`

36. **Handle AKS node pool upgrades**
    - For separate upgrades: `az aks nodepool upgrade --resource-group myResourceGroup --cluster-name myAKSCluster --name mynodepool --kubernetes-version 1.XX.Y`
    - Consider blue/green deployment with new node pools

37. **Validate Azure-specific integrations**
    - Test Azure Load Balancer functionality
    - Verify Azure CNI networking
    - Check Azure Disk and File CSI driver operation
    - Validate Azure AAD integration if used

38. **Update Terraform or Infrastructure as Code**
    - Update Terraform modules or ARM/Bicep templates to reflect the new version
    - Consider using Terraform's lifecycle meta-arguments to manage the upgrade process

## Rollback Procedures

39. **Control plane rollback**
    - For kubeadm: `kubeadm upgrade plan`
    - Identify previous version and execute `kubeadm upgrade apply v1.XX.Y` to previous version
    - For AKS: Use Azure CLI to downgrade if within supported version skew

40. **Worker node rollback**
    - Cordon and drain nodes
    - Reinstall previous kubelet version
    - Restart kubelet service
    - Uncordon nodes

41. **Etcd data restoration**
    - Stop the API server
    - Restore from snapshot: 
      ```
      ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
        --data-dir /var/lib/etcd-restore \
        --name etcd-cluster-1 \
        --initial-cluster etcd-cluster-1=https://[127.0.0.1]:2380 \
        --initial-cluster-token etcd-cluster-token \
        --initial-advertise-peer-urls https://[127.0.0.1]:2380
      ```
    - Update etcd configurations to use restored data directory
    - Restart etcd and API server

42. **Application state recovery**
    - Apply backed-up resource definitions
    - Verify application functionality
    - Restore any PVC data if needed

## Automation Opportunities

43. **Create upgrade automation scripts**
    - Develop Terraform modules for upgrading AKS clusters
    - Create PowerShell scripts for upgrade sequence automation
    - Implement Bicep templates for declarative upgrades

44. **Implement CI/CD pipeline for upgrades**
    - Create pipeline stages for each upgrade step
    - Implement automatic testing between stages
    - Add manual approval gates for production progression

45. **Develop custom controllers for upgrade orchestration**
    - Create Kubernetes operators to manage upgrade flow
    - Implement custom resource definitions for upgrade state tracking
    - Develop automated rollback triggers based on monitoring metrics