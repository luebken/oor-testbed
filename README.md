# TestBed for OOR

This is a tiny repo to discuss options for all things Observe-Only Resources. 

Issues:
* https://github.com/crossplane/crossplane/issues/1722
* https://github.com/crossplane/crossplane/issues/2099

## Paused annotation

By using the `paused` annotation (see [docs > concepts > managed-resources](https://crossplane.io/docs/v1.10/concepts/managed-resources.html#pausing-reconciliations)) we can pause the reconciliation of a specific MR. The idea here is to specify the 

### Prerequisites

* Crossplane > 1.10
* A provider that uses https://github.com/crossplane/crossplane-runtime/commit/84e629b9589852df1322ff1eae4c6e7639cf6e99


### Simple S3 Example

Using Crossplane v1.10.1 and official AWS provider v0.18.0:
```shell
helm list -n crossplane-system
NAME      	NAMESPACE        	REVISION	UPDATED                             	STATUS  	CHART            	APP VERSION
crossplane	crossplane-system	1       	2022-11-02 18:23:41.701048 +0100 CET	deployed	crossplane-1.10.1	1.10.1

kubectl get providers
NAME                   INSTALLED   HEALTHY   PACKAGE                                        AGE
upbound-provider-aws   True        True      xpkg.upbound.io/upbound/provider-aws:v0.18.0   3h47m
```

Import an existing bucket including the `crossplane.io/paused` annotation:
```shell
$ aws s3 ls
2022-11-02 22:02:42 test-bucket-18810

$ cat <<EOF | kubectl apply -f -
> apiVersion: s3.aws.upbound.io/v1beta1
> kind: Bucket
> metadata:
>   name: test-bucket-18810
>   annotations:
>     crossplane.io/external-name: test-bucket-18810
>     crossplane.io/paused: "true"
> spec:
>   forProvider:
>     region: us-east-1
> EOF
bucket.s3.aws.upbound.io/test-bucket-18810 created
```

The ready state is unknown and synced is false:
```
$ kubectl get bucket
NAME                READY   SYNCED   EXTERNAL-NAME       AGE
test-bucket-18810           False    test-bucket-18810   4m11s
```

Reconciliation is paused:
```
$ kubectl describe bucket test-bucket-18810
....
Status:
  At Provider:
  Conditions:
    Last Transition Time:  2022-11-02T21:11:26Z
    Reason:                ReconcilePaused
    Status:                False
    Type:                  Synced
Events:
  Type    Reason                Age                  From                                            Message
  ----    ------                ----                 ----                                            -------
  Normal  ReconciliationPaused  5m1s (x2 over 5m1s)  managed/s3.aws.upbound.io/v1beta1, kind=bucket  Reconciliation is paused via the pause annotation
```

This bucket is now not managed by Crossplane but you can now use it in composite and also reference to it from other MRs.

`Synced` condition at status false with reason `ReconcilePaused` as reconciliations are paused, we do not sync the resource (no external API calls are made to observe or to modify the associated external resource).

Caveat: These resources updated. For true Observe-Only-Resources a user would want to have the external state synced into the resource.


## Provider-terraform

With [provider-terraform](https://github.com/crossplane-contrib/provider-terraform) Crossplane has way of calling Terraform code directly. We can use this to re-use Terraforms [data source](https://developer.hashicorp.com/terraform/language/data-sources).

See https://github.com/crossplane-contrib/provider-terraform/tree/master/examples/observe-only-composition for a complete example. 