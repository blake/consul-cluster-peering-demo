# Test Cluster Peering Service Mesh

This uses mesh gateways for the dataplane between two peered clusters. TODO: Add
intentions and un-peering.

1. Create 2 Kubernetes clusters with a flat network. For example the terraform
   [here](https://github.com/hashicorp/consul-k8s/tree/main/charts/consul/test/terraform/eks)
   has VPC peered clusters.
1. Use `peer1-consul-values.yml` to install the Helm chart on the first cluster,
   and `peer2-consul-values.yml` on the second cluster. Use the Helm chart from the
   [peering branch](https://github.com/hashicorp/consul-k8s/tree/peering) of
   consul-k8s (This will be
   merged into main and released soon). The images being used are currently built off of
   `main` for hashicorp/consul and `peering` for hashicorp/consul-k8s.
1. Apply acceptor.yml to the first cluster:

   ```shell-session
   k apply -f acceptor.yml
   ```

1. This will cause Consul to generate a peering token called `api-token`. Get
   that token.

   ```shell-session
   k get secret api-token --output json | jq 'del(.metadata.ownerReferences)' > api-token.json
   ```

1. Apply `api-token.json` to the second cluster:

   ```shell-session
   k apply -f api-token.json
   ```

1. Apply the dialer resource, backend application, and exported service CR:

   ```shell-session
   k apply -f dialer.yml -f backend.yml -f exported-svc.yml
   ```

1. Go back to the first cluster and confirm that the peering connection is up by
   checking if the endpoints of backend deployed in peer api show up on the
   first cluster. (Need to exec onto a consul server):

   ```shell-session
   curl "localhost:8500/v1/health/connect/backend?peer=api&pretty"
   ```

1. Deploy frontend, which has an annotation describing it's explicit upstream
   backend:

   ```shell-session
   k apply -f frontend.yml
   ```

1. Exec onto frontend, and try to reach backend. The local listener for the
    upstream is `localhost:1234`, it should return successfully:

   ```shell-session
   curl localhost:1234
   ```
